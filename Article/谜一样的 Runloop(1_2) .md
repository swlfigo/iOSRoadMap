> 原文地址 http://zxfcumtcs.github.io/2014/11/15/runloop/

希望通过本文的介绍，大家能更清晰地认识 runloop。本文是 runloop 系列的第一篇文章，主要介绍 runloop 的基本概念及其与线程的关系，并试图通过 pseudo-code 的方式探索 runloop 的内部实现机制。

© 原创文章，转载请注明出处！

# Overview

* * *

刚开始接触 iOS 时，block 和 runloop 让我望而生畏。block 那神奇的语法以及与外部变量纠缠不清的关系，还有 retain cycle 这个恶魔，短时间确实难以接受！runloop 更是谜一样的虚无飘渺，又似乎若隐若现，仿佛能感受到它的存在，但始终又不得其真面目！

Apple 官方给 run loop 的定义：run loop 是线程的基础支撑，是循环处理事件的机制，一个具体的 run loop 就是一个事件处理循环。

run loop 的目的是使线程在没有事情可做时进入休眠状态，避免 CPU 空转。

> Run loops are part of the fundamental infrastructure associated with threads. A run loop is an event processing loop that you use to schedule work and coordinate the receipt of incoming events. The purpose of a run loop is to keep your thread busy when there is work to do and put your thread to sleep when there is none.

# Runloop Modes

* * *

简单地说，runloop mode 是事件源的集合 + runloop 观察者的集合。runloop 每次都运行在某个特定的 mode 上。

之所以要引入 mode 的概念，是希望 runloop 在监听过程中过滤掉不关心的事件源，只专注于某些特定的事件。

Cocoa and Core Foundation 已经为我们预定义了若干 mode，而我们也可以自定义 mode。

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-23-053955.jpg)

如果某个 input source 所属的 mode 不是当前监听的 mode，那其产生的所有事件都将被 hold 住，直到 runloop 运行在与其匹配的 mode 上。

注意：当有 UI 滑动事件时，系统会将 main runloop 切换到`NSEventTrackingRunLoopMode`，以限定此时的事件源，确保滑动的流畅性。

同时，在默认情况下`NSRunLoopCommonModes`包含`NSEventTrackingRunLoopMode`，也就是说与`NSRunLoopCommonModes`关联的事件源也与`NSEventTrackingRunLoopMode`关联。

而此时，如果有子线程想通过`performSelecorOnMainThread...`或`dispatch_async(dispatch_get_main_queue(),^{})`在主线程上执行某 `selector`，默认情况下上述两种方式产生的事件是关联到`NSRunLoopCommonModes`，因此在 UI 滑动时也会响应该事件并执行指定的 `selector`，从而影响滑动的流畅性。

为了避免此问题，可以封装上述接口，使其指定的 `selector` 运行在 main runloop 的其他 mode 上，如：`NSDefaultRunLoopMode`。

# Runloop and Thread

* * *

如果要问 runloop 与 thread 是什么关系？

答：runloop 是 thread 不断跳动的心脏！

每个 thread 都有自己的 runloop，可以通过`NSRunLoop`的类方法`currentRunLoop`获取当前线程的 runloop。但只有 main thread 的 runloop 默认是开启的，其他线程如果希望持续存活下去，就需要手动开启 runloop。

当线程没有开启 runloop 时，其生命周期将随其 `main` 函数的结束而终结；当开启 runloop 时，线程会进入休眠状态，直到相应的事件到来。形象地说前者生命周期是条直线，后者是个圆。

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-23-054028.png)

那么什么情况下需要开启线程的 runloop 呢？(当然，这里讨论的对象是子线程)

基本原则是：**线程需要处理异步事件。**

如果线程只是用来处理同步任务，没必要开启 runloop，也就是说 runloop 一定是与异步事件相关的！

具体来讲大概以下几种情况需要开启 runloop：

*   需要通过端口或自定义输入源与其他线程通讯；
*   在线程中需要使用定时器；
*   在线程上使用`performSelector`系列方法；
*   需要线程周期性地执行一些任务。

至于如何开启子线程的 runloop 就不过多的讲了，可以参考一下 Apple 的官方文档:[Threading Programming Guide](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html)

需要注意的是，在开启 runloop 前，至少要绑定一个事件源到 runloop 上，否则 runloop 将直接退出。

事件源大概有四类：

*   Port-Based Sources
*   Custom Input Sources
*   Cocoa Perform Selector Sources
*   Timer Sources

严格来讲，前三种事件源自来线程外部，Apple 将其称之为 input sources，timer 是来自线程内部的，与前三者有所区别。

对于上述事件源有几个要注意的点：

*   在子线程上执行 performSelector 时，子线程需要开启 runloop，不然 selector 将会入队，直到 runloop 启动才会被执行；
*   所有入队的 perform selector 将会在一次 runloop 中全部被处理，而不是每次 runloop 处理一个 selector；
*   当 selector 执行后，该 perform source 会被从 runloop 上移除掉；
*   因 timer 触发的事件不会使 runloop 退出，(port-based input source even 会使 runloop 退出)；
*   当 timer 触发时 runloop 正在处理其他事件，timer handler 需要等到下一个 loop 才会被执行；
*   如果 runloop 没有启动，timer 永远不会触发。

线程、runloop 以及 event source 之间的关系 ([来自](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html))：

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-23-054043.png)



# NSRunLoop Internals

* * *

`NSRunLoop`向外提供了多个接口可以启动 runloop，详细描述如下表所示：
![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-23-054122.png)



可以看到，`run`以及`runUntilDate:`内部都是通过调用`runMode:beforeDate:`函数实现的。

_对于熟悉服务器网络编程的同学来说，后台服务器其实就是一个大 runloop，其主要通过 `select`、`pool` 或 `epool` 等方式实现这个 runloop。以`select`为例，系统内核不断轮询该`select`关注的 fd，当有网络事件需要处理时，唤醒服务器进程处理事件，否则不断轮询。_

[Mike Ash](https://mikeash.com/pyblog/friday-qa-2010-01-01-nsrunloop-internals.html) 大神通过`select`，以 pseudo-code 的方式描述了`runMode:beforeDate:`的实现，同时也形象地说明了 runloop 的实质：



```objective-c
- (BOOL)runMode:(NSString *)mode beforeDate:(NSDate *)limitDate
{
   if(![self hasSourcesOrTimersForMode:mode])
      return NO;
        
   // with timer support, this code has to loop until an input source fires
   BOOL didFireInputSource = NO;
   while(!didFireInputSource)
   {
       fd_set fdset;
       FD_ZERO(&fdset);
       
       for(inputSource in [_inputSources objectForKey: mode])
           FD_SET([inputSource fileDescriptor], &fdset);
       
       // the timeout needs to be set from the limitDateand from the list of timers
       // start with the limitDate
       NSTimeInterval timeout = [limitDate timeIntervalSinceNow];
       
       // now run through the list of timers and set the
       // timeout to the smallest one found in them and
       // in the limitDate
       for(timer in [_timerSources objectForKey: mode])
           timeout = MIN(timeout, [[timer fireDate] timeIntervalSinceNow]);
       
       // now run select
       select(fdset, timeout);
       
       // process input sources first (this choice is arbitrary)
       for(inputSource in [[[_inputSources objectForKey: mode] copy] autorelease])
           if(FD_ISSET([inputSource fileDescrptor], &fdset))
           {
               didFireInputSource = YES;
               [inputSource fileDescriptorIsReady];
           }
       
       // now process timers
       // responsibility for updating fireDate for repeating timers
       // and for removing the timer from the runloop for non-repeating timers
       // rests in the timer class, not in the runloop
       for(timer in [[[_timerSources objectForKey: mode] copy] autorelease])
           if([[timer fireDate] timeIntervalSinceNow] <= 0)
               [timer fire];
       
       // see if we timed out, if so, abort!
       // this is checked at the end to ensure that timers and inputs are
       // always processed at least once before returning
       if([limitDate timeIntervalSinceNow] < 0)
           break;
   }
   
   return YES;
}
```





在上述`runMode:beforeDate:`的 pseudo-code 中：

*   第 3 行，首先判断是否有 input source 或 timer 绑定到 runloop 的指定 mode 下；
*   第 13 行，用指定 mode 下的 inputsource 设置 fdset；
*   第 23~24 行，找出此次轮询的最小过期时间；
*   第 27 行，启动 runloop；
*   第 30 行，select 返回，表明有 inputsource 到达或 timer fire；
*   30 行以下代码用于处理到达的事件，需要注意的时：如果 select 是因为 inputsource 到达而返回的，第 33 行会将`didFireInputSource`设为 YES，从而导致`runMode:beforeDate:`退出，而 timer fire 时没有设该标志位，也就是所谓的 timer fire 不会导致 runloop 退出。

是的，runloop 其实就这么简单。当然，由于没有 runloop 的源码，其内部真正如何实现不得而知。但总体思路应该大同小异！