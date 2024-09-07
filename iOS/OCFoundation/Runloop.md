#  Runloop

## 定义

runloop是一个事件驱动的大循环，它会把来自用户的交互事件、系统内部事件、计时器事件加入到事件队列中，并循环地从事件队列中取出事件进行处理，当所有的事件都处理完毕时，就会进入休眠状态，直到被新到来的事件唤醒。



**RunLoop是通过内部维护的事件循环来对事件/消息进行管理的一个对象**

**事件循环指,没有消息需要处理时,休眠以避免资源占用,有消息需要处理时,立刻唤醒**

**Runloop 的目的是使线程在没有事情可做时进入休眠状态，避免 CPU 空转。**

![img](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-23-053637.png)

- 用户态: 应用程序一般都运行在用户态上
- 内核态: 系统调用，需要使用到一些操作系统以及一些底层内核指令或者API



## 源码

通常所说的RunLoop指的是NSRunloop或者CFRunloopRef，CFRunloopRef是纯C的函数，而NSRunloop仅仅是CFRunloopRef的OC封装，并未提供额外的其他功能

```c
int32_t __CFRunLoopRun( /** 5个参数 */ )
{
    // 通知即将进入runloop
    __CFRunLoopDoObservers(KCFRunLoopEntry);
    
    do
    {
        // 通知将要处理timer和source
        __CFRunLoopDoObservers(kCFRunLoopBeforeTimers);
        __CFRunLoopDoObservers(kCFRunLoopBeforeSources);
        
        // 处理非延迟的主线程调用
        __CFRunLoopDoBlocks();
        // 处理Source0事件
        __CFRunLoopDoSource0();
        
        if (sourceHandledThisLoop) {
            __CFRunLoopDoBlocks();
         }
        // 如果有 Source1 (基于port) 处于 ready 状态，直接处理这个 Source1 然后跳转去处理消息。
        if (__Source0DidDispatchPortLastTime) {
            Boolean hasMsg = __CFRunLoopServiceMachPort();
            if (hasMsg) goto handle_msg;
        }
            
        // 通知 Observers:没有事件要处理， RunLoop 的线程即将进入休眠(sleep)。
        if (!sourceHandledThisLoop) {
            __CFRunLoopDoObservers(runloop, currentMode, kCFRunLoopBeforeWaiting);
        }
            
        // GCD dispatch main queue
        CheckIfExistMessagesInMainDispatchQueue();
        
        // 即将进入休眠
        __CFRunLoopDoObservers(kCFRunLoopBeforeWaiting);
        
        // 等待内核mach_msg事件
        mach_port_t wakeUpPort = SleepAndWaitForWakingUpPorts();
        
        // 等待。。。
        
        // 从等待中醒来
        __CFRunLoopDoObservers(kCFRunLoopAfterWaiting);
        
        // 处理因timer的唤醒
        if (wakeUpPort == timerPort)
            __CFRunLoopDoTimers();
        
        // 处理异步方法唤醒,如dispatch_async
        else if (wakeUpPort == mainDispatchQueuePort)
            __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__()
            
        // 处理Source1
        else
            __CFRunLoopDoSource1();
        
        // 再次确保是否有同步的方法需要调用
        __CFRunLoopDoBlocks();
        
    } while (!stop && !timeout);
    
    // 通知即将退出runloop
    __CFRunLoopDoObservers(CFRunLoopExit);
}

```

#### 输入源source

输入源是指事件的来源，输入源将事件异步传送到您的线程。事件的来源取决于输入源的类型，通常是两个类别之一。基于端口的输入源监视应用程序的 Mach 端口。自定义输入源监视自定义事件源。基于端口的源由内核自动发出信号，自定义源必须从另一个线程手动发出信号。 来看一下官方 Runloop 结构图（注意下图的 Input Source Port 和前面流程图中对应Source1。Source1和Timer都属于端口事件源，不同的是所有的Timer都共用一个端口“Mode Timer Port”，而每个Source1都有不同的对应端口）：

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-02-25-092153.jpg)



> **source1和source0的区别：** **source1:** 基于mach_Port的,来自系统内核或者其他进程或线程的事件，可以主动唤醒休眠中的RunLoop（iOS里进程间通信开发过程中我们一般不主动使用）。mach_port大家就理解成进程间相互发送消息的一种机制就好, 比如屏幕点击, 网络数据的传输都会触发sourse1。 苹果创建用来接受系统发出事件，当手机发生一个触摸，摇晃或锁屏等系统，这时候系统会发送一个事件到app进程（进程通信），这也就是为什么叫基于port传递source1的原因; **source0** :非基于Port的 处理事件，什么叫非基于Port的呢？就是说你这个消息不是其他进程或者内核直接发送给你的。一般是APP内部的事件, 比如hitTest:withEvent的处理, performSelectors的事件. 简单举个例子：一个APP在前台静止着，此时，用户用手指点击了一下APP界面，那么过程就是下面这样的： 我们触摸屏幕,先摸到硬件(屏幕)，屏幕表面的事件会被IOKit先包装成Event,通过mach_Port传给正在活跃的APP , Event先告诉source1（mach_port）,source1唤醒RunLoop, 然后将事件Event分发给source0,然后由source0来处理。

常见的几种源有**基于端口的源**、**自定义的源**、**performSelect源**和**计时器源；**



**RunLoop就是线程中的一个循环，RunLoop在循环中会不断检测，通过Input sources（输入源）和Timer sources（定时源）两种来源等待接受事件；然后对接受到的事件通知线程进行处理，并在没有事件的时候进行休息。**



## Runloop对象



### iOS中Runloop的API

- **Foundation**: `NSRunLoop`
- **Core Foundation**: `CFRunLoopRef`

> `NSRunLoop` 和 `CFRunLoopRef`都代表**Runloop**对象，`NSRunLoop`是基于`CFRunLoopRef`的一层OC包装，`CFRunLoopRef`是[开源的](https://link.juejin.cn?target=https%3A%2F%2Fopensource.apple.com%2Ftarballs%2FCF%2F)





* source0
  需要手动唤醒线程
* source1
  具备唤醒线程的能力





### Runloop对象的获取

- **Foundation** `[NSRunloop currentRunLoop];`获得当前线程的RunLoop对象 `[NSRunLoop mainRunLoop];`获得主线程的Runloop对象
- **Core Foundation** `CFRunLoopGetCurrent();`获得当前线程的RunLoop对象 `CFRunLoopGetMain();`获得主线程的Runloop对象



### Runloop的结构

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



**CFRunLoopRef**——这个就是Runloop对象

**CFRunLoopModeRef**——其内部主要包括四个容器，分别用来存放`source0`、`source1`、`observer`以及`timer`

**CFRunLoopSourceRef**——分为`source0`和`source1` `source0`：包括 触摸事件处理、`[performSelector: onThread: ]` `source1`：包括 基于Port的线程间通信、系统事件捕捉

**CFRunLoopTimerRef**——`timer`事件，包括我们设置的定时器事件、`[performSelector: withObject: afterDelay:]`

**CFRunLoopObserverRef**——监听者，**Runloop**状态变更的时，会通知监听者进行函数回调，UI界面的刷新就是在监听到Runloop状态为`BeforeWaiting`时进行的。

上这几个类相互之间的关系，可以通过如下的图来描绘.

![img](https://sylarimage.oss-cn-shenzhen.aliyuncs.com/uPic/a8e3b590d3de447f9797def60d86e096~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)



一个**RunLoop**对象里面包含了若干个`RunLoopMode`，**RunLoop**内部是通过一个集合容器`_modes`来装这些`RunLoopMode`的。

**RunLoopMode**内部核心内容是4个数组容器，分别用来装`source0`，`source1`，`observer`和`timer`，**RunLoop**对象内部有一个`_currentMode`，它指向了该RunLoop对象的其中一个`RunLoopMode`，它代表的含义是RunLoop当前所运行的`RunLoopMode`，所谓“运行”也就是说，RunLoop当前只会执行`_currentMode`所指向的`RunLoopMode`里面所包括的事件（`source0、source1、observer、timer`）



**RunLoop启动时只能选择其中一个Mode作为currentMode。**



还有就是**RunLoop**对象内部还包括一个线程对象`_pthread`，这就是跟它一一对应的那个线程对象。



<img src="http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-07-23-134756.jpg" style="zoom:25%;" />



<img src="http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-07-23-134804.jpg" style="zoom:25%;" />



#### source0

包括**触摸事件处理**、`[performSelector: onThread: ]`



#### source1

**source1**包括系统事件捕捉和基于`port`的线程间通信。什么是**系统事件捕捉**？又如何理解**基于`port`的线程间通信**？其实，我们手指点击屏幕，首先产生的是一个系统事件，通过`source1`来接受捕捉，然后由**Springboard**程序包装成`source0`分发给应用去处理，因此我们在App内部接受到触摸事件，就是`source0`，

<img src="https://sylarimage.oss-cn-shenzhen.aliyuncs.com/uPic/ecd90a8b62674b679a171ecd40dc4f03~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp" alt="img" style="zoom: 67%;" />

**基于port的线程间通信**通过下面的图示大致理解即可

<img src="https://sylarimage.oss-cn-shenzhen.aliyuncs.com/uPic/c3ae1444230c44889f6cac0ba0d6fda9~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp" alt="image.png" style="zoom:67%;" />

#### 

#### CFRunLoopTimerRef

同样，可以在Xcode里面通过**LLDB**的`bt`指令，查看`NSTimer`事件和`[performSelector: withObject: afterDelay]`事件的函数调用栈，发现它们都是通过 `__CFRUNLOOP_IS_CALLING_OUT_TO_A_TIMER_CALLBACK_FUNCTION__`函数被吊起的。从函数名看出，它们确实是属于**timer**事件（`CFRunLoopTimerRef`）



#### CFRunLoopObserverRef

我们知道  `observer`  是用来监听**Runloop**状态的。还可以处理UI界面刷新，那我们些的那些UI界面相关的控制代码，是怎么被执行的呢？图示如下

![img](https://sylarimage.oss-cn-shenzhen.aliyuncs.com/uPic/4c78dfc2b9c3487592e558df0e745f15~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) 



**Runloop状态**总共有以下几种

```c
/* Run Loop Observer Activities */
typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {
    kCFRunLoopEntry = (1UL << 0),//进入runloop循环
    kCFRunLoopBeforeTimers = (1UL << 1),//即将处理timer事件
    kCFRunLoopBeforeSources = (1UL << 2),//即将处理source事件
    kCFRunLoopBeforeWaiting = (1UL << 5),//即将进入休眠（等待消息唤醒）
    kCFRunLoopAfterWaiting = (1UL << 6),//休眠结束（被消息唤醒）
    kCFRunLoopExit = (1UL << 7),//退出runloop循环
    kCFRunLoopAllActivities = 0x0FFFFFFFU//集合以上所有的状态
};
```



#### _modes和_commonModes

`Runloop Mode` 是`事件源的集合` + `Runloop`观察者的集合。**Runloop 每次都运行在某个特定的 mode 上。**

之所以要引入 mode 的概念，是**希望 Runloop 在监听过程中过滤掉不关心的事件源，只专注于某些特定的事件。**

 `Runloop`总是运行在某种特定的CFRunLoopModeRef下,意思是每次`Runloop`开始时候会选择一个mode，执行这个mode里面的 `block`,`timer`等事件.这可以解释滑动过程中，`NSTimer`为什么会停止,因为滑动过程中`Runloop`处于 `TrackingMode`,`NSTimer`默认添加在`DefaultMode`,所以不执行



如果某个 `input source` 所属的 mode 不是当前监听的 mode，那 **其产生的所有事件都将被 hold 住，直到 runloop 运行在与其匹配的 mode 上



系统默认注册了5个Mode:

1. `kCFRunLoopDefaultMode`: App的默认 Mode，通常主线程是在这个 Mode 下运行的。
2. `UITrackingRunLoopMode`: 界面跟踪 Mode，用于 ScrollView 追踪触摸滑动，保证界面滑动时不受其他 Mode 影响。
3. `UIInitializationRunLoopMode`: 在刚启动 App 时第进入的第一个 Mode，启动完成后就不再使用。
4. `GSEventReceiveRunLoopMode`: 接受系统事件的内部 Mode，通常用不到。
5. `kCFRunLoopCommonModes`: 伪模式，这是一个占位的 Mode，没有实际作用。



##### CommonMode的特殊性

`NSRunLoopCommonModes`

- CommonMode不是实际存在的一种Mode
- 是同步Source/Timer/Observer到多个Mode中的一种技术方案





![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-07-23-134817.png)

`NSRunLoopCommonModes`其实不是一个具体的模式，它可以理解成一个标签，被打上这种标签的具体Mode会被放入到**RunLoop**内部的一个容器成员`_commonModes `里面，它是一个`CFMutableSetRef`，默认情况下，`_commonModes `内部装着`kCFRunLoopDefaultMode ` + `UITrakingRunLoopMode `这两个Mode，等于说这两个Mode是具有`NSRunLoopCommonModes`标记的，因此都被添加进了`_commonModes `，根据上面的代码，`timer`将不会被添加到某个具体的Mode里，而是会被放入**RunLoop**的`_commonModeItems `这个容器里。只要App运行在`_commonModes `所包含的某个Mode下，就会去处理`_commonModeItems `里面的事件。当然，所运行的那个Mode自己本身所包含的事件也是会被处理的，

如果有子线程想通过`performSelecorOnMainThread...` 或 `dispatch_async(dispatch_get_main_queue(),^{})` 在主线程上执行某 selector，默认情况下上述两种方式产生的事件是关联到`NSRunLoopCommonModes`，因此在 UI 滑动时也会响应该事件并执行指定的 selector，从而影响滑动的流畅性。





### Runtloop运行流程

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



### Runloop & Thread

`Runloop`与线程是一一对应的，一个runloop对应一个核心的线程。每个 `thread` 都有自己的 `Runloop`，可以通过 `NSRunLoop`的类方法`currentRunLoop`获取**当前线程**的 `runloop`。

但只有 `main thread` 的 `runloop` **默认是开启**的，**其他线程如果希望持续存活下去，就需要手动开启Runloop**。

对于子线程来说，runloop是懒加载的，只有当我们使用的时候才会创建，

所以在子线程用定时器要注意：确保子线程的runloop被创建，不然定时器不会回调。

`Runloop`是来管理线程的，当线程的runloop被开启后，线程会在执行完任务后进入休眠状态，有了任务就会被唤醒去执行任务。



### RunLoop相关类

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



### 总结

* main runloop在主界面即将显示前由系统启动(主界面 controller 的 viewWillAppear:执行后启动)；

* runloop 启动后(唤醒后)会依次处理 timer(如果有)、source event(如果有)并在此前通知 observer；

* main runloop每分钟会被唤醒一次

* UI事件唤醒 main runloop 直到处理完该事件，如果该事件含有异步操作，runloop 不会等待异步操作完成；

* UIViewController的viewWillAppear:和viewDidAppear:不在同一次 runloop 中被调用；

* timer会唤醒 runloop 但不会使 runloop 退出；

* 如果子线程的 runloop 没有绑定 timer 或 source event，其 runloop 不会启动；

* 一次 runloop 可以处理多个事件。



### 注意点

#### Timer Sources(NSTimer)

Timer source 会在未来一个预定时间向线程同步分发事件。线程可以用 Timer 来通知自己做一些事情。比如用户在搜索栏输入一连串字符之后的某个时间自动搜索一次结果。正是因为有了个延时，才让用户有机会在自动搜索发生前尽可能打出想要的搜索字符串。

**Timer 并不是实时的，会有误差**。如果一个 timer 不在正在运行的 runloop 监控的 mode 中，需要一直等到 runloop 运行在一个支持这个 timer 的 mode 时，timer 才会触发。**如果一个 timer 触发的时候恰巧 run loop 正忙于执行某个 handler 程序，这个 timer 的 handler 程序需要等到下次才会通过 run loop 执行。**如果 runloop 根本不在运行，timer 永远都不会触发。

可以配置 timer 只生成一次或重复多次事件。重复的 timer 每次会根据已经编排的触发时间自动重新编排。如果实际的触发时间太过于延迟，甚至是晚了一个或多个周期，那么也只会触发一次，而非连续多次。**之后会重新编排下次触发时间**。



### Runloop 运用场景

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



### RunLoop中Source0和Source1的区别

`Source0`并不能主动触发事件。使用时，你需要先调用`CFRunLoopSourceSignal`，将这个Source标记为待处理，然后手动调用`CFRunLoopWakeUp`来唤醒RunLoop，让其处理这个事件。

`Source1`能主动触发事件。其中它有一个`mach_port_t`，mach_port是用于内核向线程发送消息的。

使用`Source0`的情况：

- 触摸事件处理；
- 调用`performSelector:onThread:withObject:waitUntilDone:`方法；

使用`Source1`的情况：

- 基于端口的线程间通信（A线程通过端口发送消息到B线程，这个消息是`Source1`的；
- 系统事件的捕捉，先触发是`Source1`，接着分发到`Source0`去处理。

### RunLoop响应用户操作

以按钮点击触发事件为例，点击屏幕的时候，首先系统内部捕获到这个点击事件，这是在`Source1`中处理的，`Source1`会包装成事件丢到事件队列中，交给`Source0`处理。



### RunLoop与UI刷新

当UI需要更新的时候，比如改变了`frame`、更新了`UIView`/`CALayer`的层次时，或者手动调用了`setNeedsLayout`/`setNeedsDisplay`方法后，这个`UIView`/`CALayer`就被标记为待处理，并被提交到一个全局的容器去。

苹果注册了一个`Observer`监听`BeforeWaiting`(即将进入休眠) 和 `Exit`(即将退出Loop) 事件，回调去执行一个很长的函数：`CA::Transaction::observer_callback(__CFRunLoopObserver*, unsigned long, void*)()` 这个函数里会遍历所有待处理的`UIView`/`CAlayer`以执行实际的绘制和调整，并更新界面。

<img src="http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-03-05-032038.jpg" alt="RunLoop与UI刷新" style="zoom:50%;" />



### RunLoop与AutoreleasePool

在程序启动之后，主线程会创建一个Runloop，也会创建两个Observer，回调工作都是在`_wrapRunLoopWithAutoreleasePoolHandler`函数中。

第一个Observer监听的是Entry（即将进入Loop），回调是在`_objc_autoreleasePoolPush()`中创建自动释放池的，优先级是最高的，保证创建释放池是在所有回调之前。

第二个Observer监听有两个事件：BeforeWaiting（进入休眠）时调用`_objc_autoreleasePoolPop`和`_objc_autoreleasePoolPush`释放旧的释放池以及创建新的释放池；Exit（退出Loop）调用`_objc_autoreleasePoolPop`来释放自动释放池。这个优先级是最低的，保证释放池发生在所有回调之后调用。





## Reference

[1.谜一样的 Runloop](http://zxfcumtcs.github.io/2014/11/15/runloop/)

[2.深入理解RunLoop](https://blog.ibireme.com/2015/05/18/runloop/)

[3.OC中的RunLoop](https://juejin.cn/post/6930891193748307981)

[Runloop的内部结构与运行原理什么是Runloop Runloop顾名思义，就是运行循环。首先它根程序运行过程有关系 - 掘金 (juejin.cn)](https://juejin.cn/post/6965790003951566861)
