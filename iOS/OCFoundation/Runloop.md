#  Runloop

NSRunLoop是CFRunLoop的封装,提供了面向对象的API

Apple 官方给 `Runloop` 的定义：
Runloop 是线程的基础支撑，是循环处理事件的机制，一个具体的 Runloop 就是一个事件处理循环。

**RunLoop是通过内部维护的事件循环来对事件/消息进行管理的一个对象**

**事件循环指,没有消息需要处理时,休眠以避免资源占用,有消息需要处理时,立刻唤醒**

**Runloop 的目的是使线程在没有事情可做时进入休眠状态，避免 CPU 空转。**

> Run loops are part of the fundamental infrastructure associated with threads. A run loop is an event processing loop that you use to schedule work and coordinate the receipt of incoming events. The purpose of a run loop is to keep your thread busy when there is work to do and put your thread to sleep when there is none.



![img](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-23-053637.png)

- 用户态: 应用程序一般都运行在用户态上
- 内核态: 系统调用，需要使用到一些操作系统以及一些底层内核指令或者API



一般来讲，一个线程一次只能执行一个任务，执行完成后线程就会退出。如果我们需要一个机制，让线程能随时处理事件但并不退出，这种模型通常被称作 Event Loop。 iOS里的`RunLoop`也是。

实现这种模型的关键点在于：如何管理事件/消息，如何让线程在没有处理消息时休眠以避免资源占用、在有消息到来时立刻被唤醒。

`RunLoop`实际上是一个对象，这个对象在循环中用来处理程序运行过程中出现的各种事件（比如说触摸事件、UI刷新事件、定时器事件、Selector事件），从而保持程序的持续运行；而且在没有事件处理的时候，会进入睡眠模式，从而节省CPU资源，提高程序性能。

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-02-25-092153.jpg)



**RunLoop就是线程中的一个循环，RunLoop在循环中会不断检测，通过Input sources（输入源）和Timer sources（定时源）两种来源等待接受事件；然后对接受到的事件通知线程进行处理，并在没有事件的时候进行休息。**



## iOS中有两套API来访问和使用Runloop 

Foundation： NSRunLoop

Core Foundation : CFRunLoopRef

#### Runloop的结构

```cpp
struct __CFRunLoop {
    CFRuntimeBase _base;
    pthread_mutex_t _lock;			/* locked for accessing mode list */
    __CFPort _wakeUpPort;			// used for CFRunLoopWakeUp 
    Boolean _unused;
    volatile _per_run_data *_perRunData;              // reset for runs of the run loop
    pthread_t _pthread;
    uint32_t _winthread;
    CFMutableSetRef _commonModes;
    CFMutableSetRef _commonModeItems;	
    CFRunLoopModeRef _currentMode;		//当前Mode
    CFMutableSetRef _modes;			//所有mode的集合
    struct _block_item *_blocks_head;
    struct _block_item *_blocks_tail;
    CFAbsoluteTime _runTime;
    CFAbsoluteTime _sleepTime;
    CFTypeRef _counterpart;
};
```

mode类型的`CFRunLoopModeRef`

`CFRunLoopModeRef` 其实是指向`__CFRunLoopMode`结构体的指针:

```cpp
typedef struct __CFRunLoopMode *CFRunLoopModeRef;
```



### CFRunLoop

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-07-23-134756.jpg)

###  CFRunLoopMode

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-07-23-134804.jpg)

* source0
  需要手动唤醒线程
* source1
  具备唤醒线程的能力



## 各个数据结构之间关系



一个Runloop中包含很多个Mode,Mode中包含Source,Timer与Observer

Input Source, Timer Source, Run Loop Observer 统称为 Mode Item，这里的 Mode 指的是 Run Loop Mode。一个 Run Loop 包含若干个 Mode，每个 Mode 又包含若干个 Item。Item 与 Mode 是多对多的关系，没有 Item 的 Mode 会立刻退出



**CFRunLoopModeRef代表RunLoop的运行模式**

**一个RunLoop包含若干个Mode，每个Mode又包含若干个Source0/Source1/Timer/Observer**

**RunLoop启动时只能选择其中一个Mode作为currentMode。**
![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-02-25-095932.jpg)

一个RunLoop对象（CFRunLoopRef）中包含若干个运行模式（CFRunLoopModeRef）。而每一个运行模式下又包含若干个输入源（CFRunLoopSourceRef）、定时源（CFRunLoopTimerRef）、观察者（CFRunLoopObserverRef）



系统默认注册了5个Mode:

1. `kCFRunLoopDefaultMode`: App的默认 Mode，通常主线程是在这个 Mode 下运行的。
2. `UITrackingRunLoopMode`: 界面跟踪 Mode，用于 ScrollView 追踪触摸滑动，保证界面滑动时不受其他 Mode 影响。
3. `UIInitializationRunLoopMode`: 在刚启动 App 时第进入的第一个 Mode，启动完成后就不再使用。
4. `GSEventReceiveRunLoopMode`: 接受系统事件的内部 Mode，通常用不到。
5. `kCFRunLoopCommonModes`: 伪模式，这是一个占位的 Mode，没有实际作用。



伪模式（`kCFRunLoopCommonModes`），这其实不是一种真实的模式，而是一种标记模式，意思就是可以在打上`Common Modes`标记的模式下运行。

那么哪些模式被标记上了Common Modes呢？

`NSDefaultRunLoopMode` 和 `UITrackingRunLoopMode`。



## Runloop Modes

`Runloop Mode` 是`事件源的集合` + `Runloop`观察者的集合。**Runloop 每次都运行在某个特定的 mode 上。**

之所以要引入 mode 的概念，是**希望 Runloop 在监听过程中过滤掉不关心的事件源，只专注于某些特定的事件。**



 `Runloop`总是运行在某种特定的CFRunLoopModeRef下,意思是每次`Runloop`开始时候会选择一个mode，执行这个mode里面的 `block`,`timer`等事件.这可以解释滑动过程中，`NSTimer`为什么会停止,因为滑动过程中`Runloop`处于 `TrackingMode`,`NSTimer`默认添加在`DefaultMode`,所以不执行

#### CommonMode的特殊性

`NSRunLoopCommonModes`

- CommonMode不是实际存在的一种Mode
- 是同步Source/Timer/Observer到多个Mode中的一种技术方案





![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-07-23-134817.png)

如果某个 `input source` 所属的 mode 不是当前监听的 mode，那 **其产生的所有事件都将被 hold 住，直到 runloop 运行在与其匹配的 mode 上。**



应用场景举例：主线程的 RunLoop 里有两个预置的 Mode：`kCFRunLoopDefaultMode` 和 `UITrackingRunLoopMode`。这两个 Mode 都已经被标记为”Common”属性。`DefaultMode` 是 App 平时所处的状态，`TrackingRunLoopMode` 是追踪 ScrollView 滑动时的状态。当你创建一个 Timer 并加到 `DefaultMode` 时，Timer 会得到重复回调，但此时滑动一个TableView时，**RunLoop 会将 mode 切换为**` TrackingRunLoopMode`，这时 Timer 就不会被回调，并且也不会影响到滑动操作。



同时，在默认情况下`NSRunLoopCommonModes`包含`NSEventTrackingRunLoopMode`，也就是说与`NSRunLoopCommonModes`关联的事件源也与`NSEventTrackingRunLoopMode`关联。

而此时，如果有子线程想通过`performSelecorOnMainThread...` 或 `dispatch_async(dispatch_get_main_queue(),^{})` 在主线程上执行某 selector，默认情况下上述两种方式产生的事件是关联到`NSRunLoopCommonModes`，因此在 UI 滑动时也会响应该事件并执行指定的 selector，从而影响滑动的流畅性。

为了避免此问题，可以封装上述接口，使其指定的 selector 运行在 Main Runloop 的其他 mode 上，如：`NSDefaultRunLoopMode`。



## CFRunLoopObserver 观察者
观察时间点
* kCFRunLoopEntry
* kCFRunLoopBeforeTimers
* kCFRunLoopBeforeSources
* kCFRunLoopBeforeWaiting
* kCFRunLoopAfterWaiting
* kCFRunLoopExit



## Runtloop运行流程

![img](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-07-23-134820.png)

Run Loop 接收的事件来源 (source) 有两种。

- Input Source 传送来自其他应用或线程的异步事件/消息；
- Timer Source 传送的是基于定时器的同步事件，可以定时或重复发送。

`Source0`:

- 触摸事件处理
- `performSelector: onThread:`

`Source1`:

- 基于`port`的线程通信
- 系统事件捕捉

`Timers`:

- `NSTimer`
- `performSelector:withObject:afterDelay:`

`Observers`:

- 用于监听`RunLoop`的状态
- UI刷新（BeforeWaiting）
- `Autoreleasepool`(BeforeWaiting)



## Runloop & Thread

`Runloop`与线程是一一对应的，一个runloop对应一个核心的线程。每个 `thread` 都有自己的 `Runloop`，可以通过 `NSRunLoop`的类方法`currentRunLoop`获取**当前线程**的 `runloop`。

但只有 `main thread` 的 `runloop` **默认是开启**的，**其他线程如果希望持续存活下去，就需要手动开启Runloop**。对于子线程来说，runloop是懒加载的，只有当我们使用的时候才会创建，所以在子线程用定时器要注意：确保子线程的runloop被创建，不然定时器不会回调。

`Runloop`是来管理线程的，当线程的runloop被开启后，线程会在执行完任务后进入休眠状态，有了任务就会被唤醒去执行任务。

## RunLoop相关类

Core Foundation框架下关于RunLoop的5个类：

1. CFRunLoopRef：代表RunLoop的对象
2. CFRunLoopModeRef：RunLoop的运行模式
3. CFRunLoopSourceRef：就是RunLoop模型图中提到的输入源/事件源
4. CFRunLoopTimerRef：就是RunLoop模型图中提到的定时源
5. CFRunLoopObserverRef：观察者，能够监听RunLoop的状态改变

我们可通过以下方式来获取RunLoop对象：

- Core Foundation
  - `CFRunLoopGetCurrent(); // 获得当前线程的RunLoop对象`
  - `CFRunLoopGetMain(); // 获得主线程的RunLoop对象`
- Foundation
  - `[NSRunLoop currentRunLoop]; // 获得当前线程的RunLoop对象`
  - `[NSRunLoop mainRunLoop]; // 获得主线程的RunLoop对象`



`CFRunLoopGetCurrent` :

```cpp
CFRunLoopRef CFRunLoopGetCurrent(void) {
    CHECK_FOR_FORK();
    CFRunLoopRef rl = (CFRunLoopRef)_CFGetTSD(__CFTSDKeyRunLoop);
    if (rl) return rl;
    return _CFRunLoopGet0(pthread_self());
}
```

查看`_CFRunLoopGet0`方法内部

```cpp
CF_EXPORT CFRunLoopRef _CFRunLoopGet0(pthread_t t) {
    if (pthread_equal(t, kNilPthreadT)) {
	t = pthread_main_thread_np();
    }
    __CFLock(&loopsLock);
    if (!__CFRunLoops) {
        __CFUnlock(&loopsLock);
        // 创建一个dict
	CFMutableDictionaryRef dict = CFDictionaryCreateMutable(kCFAllocatorSystemDefault, 0, NULL, &kCFTypeDictionaryValueCallBacks);
        // 根据传入的主线程获取主线程对应的RunLoop
	CFRunLoopRef mainLoop = __CFRunLoopCreate(pthread_main_thread_np());
        // 保存主线程 将主线程-key和RunLoop-Value保存到字典中
	CFDictionarySetValue(dict, pthreadPointer(pthread_main_thread_np()), mainLoop);
    
	if (!OSAtomicCompareAndSwapPtrBarrier(NULL, dict, (void * volatile *)&__CFRunLoops)) {
	    CFRelease(dict);
	}
	CFRelease(mainLoop);
        __CFLock(&loopsLock);
    }
    
    // 从字典里面拿，将线程作为key从字典里获取一个loop
    CFRunLoopRef loop = (CFRunLoopRef)CFDictionaryGetValue(__CFRunLoops, pthreadPointer(t));
    __CFUnlock(&loopsLock);

    // 如果loop为空，则创建一个新的loop，所以runloop会在第一次获取的时候创建
    if (!loop) {
	CFRunLoopRef newLoop = __CFRunLoopCreate(t);
        __CFLock(&loopsLock);
	loop = (CFRunLoopRef)CFDictionaryGetValue(__CFRunLoops, pthreadPointer(t));
    
        // 创建好之后，以线程为key runloop为value，一对一存储在字典中，下次获取的时候，则直接返回字典内的runloop
	if (!loop) {
	    CFDictionarySetValue(__CFRunLoops, pthreadPointer(t), newLoop);
	    loop = newLoop;
	}
        // don't release run loops inside the loopsLock, because CFRunLoopDeallocate may end up taking it
        __CFUnlock(&loopsLock);
	CFRelease(newLoop);
    }
    if (pthread_equal(t, pthread_self())) {
        _CFSetTSD(__CFTSDKeyRunLoop, (void *)loop, NULL);
        if (0 == _CFGetTSD(__CFTSDKeyRunLoopCntr)) {
            _CFSetTSD(__CFTSDKeyRunLoopCntr, (void *)(PTHREAD_DESTRUCTOR_ITERATIONS-1), (void (*)(void *))__CFFinalizeRunLoop);
        }
    }
    return loop;
}
```



**线程和 RunLoop 之间是一一对应的，其关系是保存在一个 Dictionary 里。所以我们创建子线程RunLoop时，只需在子线程中获取当前线程的RunLoop对象即可`[NSRunLoop currentRunLoop];`**

**如果不获取，那子线程就不会创建与之相关联的RunLoop，并且只能在一个线程的内部获取其 RunLoop `[NSRunLoop currentRunLoop];`**

**方法调用时，会先看一下字典里有没有存子线程相对用的RunLoop，如果有则直接返回RunLoop，如果没有则会创建一个，并将与之对应的子线程存入字典中。当线程结束时，RunLoop会被销毁。**



## Summary

* main runloop在主界面即将显示前由系统启动(主界面 controller 的 viewWillAppear:执行后启动)；

* runloop 启动后(唤醒后)会依次处理 timer(如果有)、source event(如果有)并在此前通知 observer；

* main runloop每分钟会被唤醒一次

* UI事件唤醒 main runloop 直到处理完该事件，如果该事件含有异步操作，runloop 不会等待异步操作完成；

* UIViewController的viewWillAppear:和viewDidAppear:不在同一次 runloop 中被调用；

* timer会唤醒 runloop 但不会使 runloop 退出；

* 如果子线程的 runloop 没有绑定 timer 或 source event，其 runloop 不会启动；

* 一次 runloop 可以处理多个事件。



## 注意点

### Timer Sources(NSTimer)

Timer source 会在未来一个预定时间向线程同步分发事件。线程可以用 Timer 来通知自己做一些事情。比如用户在搜索栏输入一连串字符之后的某个时间自动搜索一次结果。正是因为有了个延时，才让用户有机会在自动搜索发生前尽可能打出想要的搜索字符串。

**Timer 并不是实时的，会有误差**。如果一个 timer 不在正在运行的 run loop 监控的 mode 中，需要一直等到 run loop 运行在一个支持这个 timer 的 mode 时，timer 才会触发。**如果一个 timer 触发的时候恰巧 run loop 正忙于执行某个 handler 程序，这个 timer 的 handler 程序需要等到下次才会通过 run loop 执行。**如果 run loop 根本不在运行，timer 永远都不会触发。

可以配置 timer 只生成一次或重复多次事件。重复的 timer 每次会根据已经编排的触发时间自动重新编排。如果实际的触发时间太过于延迟，甚至是晚了一个或多个周期，那么也只会触发一次，而非连续多次。**之后会重新编排下次触发时间**。



## Runloop 运用场景

以`AFNetworking2.x` 保活原理来说:

```
/*
AFNetworking/NSURLConnection/AFURLConnectionOperation.m
*/ 
+ (NSThread *)networkRequestThread {
    static NSThread *_networkRequestThread = nil;
    static dispatch_once_t oncePredicate;
    dispatch_once(&oncePredicate, ^{
        _networkRequestThread = [[NSThread alloc] initWithTarget:self selector:@selector(networkRequestThreadEntryPoint:) object:nil];
        [_networkRequestThread start];
    });

    return _networkRequestThread;
}
------------------------------------------------------------------------
/*
AFNetworking/NSURLConnection/AFURLConnectionOperation.m
*/    

+ (void)networkRequestThreadEntryPoint:(id)__unused object {
    @autoreleasepool {
        [[NSThread currentThread] setName:@"AFNetworking"];

        NSRunLoop *runLoop = [NSRunLoop currentRunLoop];
        [runLoop addPort:[NSMachPort port] forMode:NSDefaultRunLoopMode];
        [runLoop run];
    }
}
```

如代码，为线程中Runloop添加一个 `[NSMachPort port]` `source1` 事件源，让线程不退出一直保活。直到 `AF3.x`,废弃了 `NSURLConnection`。因为`NSURLConnection`中,执行回调的要在子线程,可能回调回来线程已经销毁无法做回调.`3.x`版本中，使用了 `NSURLSession`,能指定`queue`回调，所以避免了问题



## RunLoop中Source0和Source1的区别

`Source0`并不能主动触发事件。使用时，你需要先调用`CFRunLoopSourceSignal`，将这个Source标记为待处理，然后手动调用`CFRunLoopWakeUp`来唤醒RunLoop，让其处理这个事件。

`Source1`能主动触发事件。其中它有一个`mach_port_t`，mach_port是用于内核向线程发送消息的。

使用`Source0`的情况：

- 触摸事件处理；
- 调用`performSelector:onThread:withObject:waitUntilDone:`方法；

使用`Source1`的情况：

- 基于端口的线程间通信（A线程通过端口发送消息到B线程，这个消息是`Source1`的；
- 系统事件的捕捉，先触发是`Source1`，接着分发到`Source0`去处理。

## RunLoop响应用户操作

以按钮点击触发事件为例，点击屏幕的时候，首先系统内部捕获到这个点击事件，这是在`Source1`中处理的，`Source1`会包装成事件丢到事件队列中，交给`Source0`处理。

## RunLoop与UI刷新

当UI需要更新的时候，比如改变了`frame`、更新了`UIView`/`CALayer`的层次时，或者手动调用了`setNeedsLayout`/`setNeedsDisplay`方法后，这个`UIView`/`CALayer`就被标记为待处理，并被提交到一个全局的容器去。

苹果注册了一个`Observer`监听`BeforeWaiting`(即将进入休眠) 和 `Exit`(即将退出Loop) 事件，回调去执行一个很长的函数：`CA::Transaction::observer_callback(__CFRunLoopObserver*, unsigned long, void*)()` 这个函数里会遍历所有待处理的`UIView`/`CAlayer`以执行实际的绘制和调整，并更新界面。

![RunLoop与UI刷新](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-03-05-032038.jpg)

## RunLoop与AutoreleasePool

在程序启动之后，主线程会创建一个Runloop，也会创建两个Observer，回调工作都是在`_wrapRunLoopWithAutoreleasePoolHandler`函数中。

第一个Observer监听的是Entry（即将进入Loop），回调是在`_objc_autoreleasePoolPush()`中创建自动释放池的，优先级是最高的，保证创建释放池是在所有回调之前。

第二个Observer监听有两个事件：BeforeWaiting（进入休眠）时调用`_objc_autoreleasePoolPop`和`_objc_autoreleasePoolPush`释放旧的释放池以及创建新的释放池；Exit（退出Loop）调用`_objc_autoreleasePoolPop`来释放自动释放池。这个优先级是最低的，保证释放池发生在所有回调之后调用。





## Reference

[1.谜一样的 Runloop](http://zxfcumtcs.github.io/2014/11/15/runloop/)

[2.深入理解RunLoop](https://blog.ibireme.com/2015/05/18/runloop/)

[3.OC中的RunLoop](https://juejin.cn/post/6930891193748307981)

