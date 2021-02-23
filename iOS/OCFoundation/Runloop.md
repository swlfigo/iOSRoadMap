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

## CFRunLoop

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-07-23-134756.jpg)

##  CFRunLoopMode

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-07-23-134804.jpg)

* source0
  需要手动唤醒线程
* source1
  具备唤醒线程的能力

## CFRunLoopObserver 观察者
观察时间点
* kCFRunLoopEntry
* kCFRunLoopBeforeTimers
* kCFRunLoopBeforeSources
* kCFRunLoopBeforeWaiting
* kCFRunLoopAfterWaiting
* kCFRunLoopExit

## 各个数据结构之间关系
![](http://img.isylar.com/media/15498936308142.jpg)
一个Runloop中包含很多个Mode,Mode中包含Source,Timer与Observer

Input Source, Timer Source, Run Loop Observer 统称为 Mode Item，这里的 Mode 指的是 Run Loop Mode。一个 Run Loop 包含若干个 Mode，每个 Mode 又包含若干个 Item。Item 与 Mode 是多对多的关系，没有 Item 的 Mode 会立刻退出



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



## Runtloop运行流程

![img](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-07-23-134820.png)

Run Loop 接收的事件来源 (source) 有两种。

- Input Source 传送来自其他应用或线程的异步事件/消息；
- Timer Source 传送的是基于定时器的同步事件，可以定时或重复发送。



## Runloop & Thread

`Runloop`与线程是一一对应的，一个runloop对应一个核心的线程。每个 `thread` 都有自己的 `Runloop`，可以通过 `NSRunLoop`的类方法`currentRunLoop`获取**当前线程**的 `runloop`。

但只有 `main thread` 的 `runloop` **默认是开启**的，**其他线程如果希望持续存活下去，就需要手动开启Runloop**。对于子线程来说，runloop是懒加载的，只有当我们使用的时候才会创建，所以在子线程用定时器要注意：确保子线程的runloop被创建，不然定时器不会回调。

`Runloop`是来管理线程的，当线程的runloop被开启后，线程会在执行完任务后进入休眠状态，有了任务就会被唤醒去执行任务。



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

## Reference

[1.谜一样的 Runloop](http://zxfcumtcs.github.io/2014/11/15/runloop/)

[2.深入理解RunLoop](https://blog.ibireme.com/2015/05/18/runloop/)

