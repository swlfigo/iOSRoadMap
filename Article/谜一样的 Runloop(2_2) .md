> 原文地址 http://zxfcumtcs.github.io/2014/11/30/runloop2/

希望通过本文的介绍，大家能更清晰地认识 runloop。本文是 runloop 系列的第二篇文章，主要介绍通过 runloop observer 来近距离的观察 runloop。

© 原创文章，转载请注明出处！

# The Run Loop Sequence of Events

* * *

下面我们看一下 runloop 从启动到退出的整个生命周期内做了哪些事情：

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-23-054440.jpg)

从上图可以看出，runloop 的生命周期大概要经历三个阶段：

*   runloop 启动后首先处理待处理的事件，如：timer、input source；
*   进入休眠状态，等待新任务的到来；
*   有新任务了，runloop(准确地说是 thread) 被唤醒，处理新任务，runloop 重启或退出。

需要注意的是：runloop 启动后如果有 port-based input sources event 需要处理，则 runloop 在处理完该事件后直接退出。

runloop 在整个生命周期内还有一件重要的事情就是与 observer 的交互。我们可以通过设置 runloop 的 observer 来监控 runloop 的内部状态。



# Understanding Runloop Internals by Runloop-Observer

* * *

runloop 之所以像谜一样让人琢磨不透，主要在于我们难于了解其内部实现机制、触发时机，同时也很难感受到它的存在。虽然，[前文](http://zxfcumtcs.github.io/2014/11/15/runloop/)通过 pseudo-code 的形式模拟了 runloop 的内部实现，但对于 runloop 真正何时会被触发，何时结束还是不太清楚。

幸运的是，Apple 还是给了我们近距离观察 runloop 的机会——runloop observer(可以形象地将其称之为 runloop 之窗 ^_^)。

通过 runloop observer 我们可以近距离的监听 runloop 以下重要事件：

*   runloop 启动——kCFRunLoopEntry；
*   runloop 即将处理 timer——kCFRunLoopBeforeTimers；
*   runloop 即将处理 input source event——kCFRunLoopBeforeSources；
*   runloop 即将进入休眠状态——kCFRunLoopBeforeWaiting；
*   runloop 被唤醒，但还未处理唤醒它的事件——kCFRunLoopAfterWaiting；
*   runloop 退出——kCFRunLoopExit。

## CFRunLoopObserver —— runloop 之窗。

Apple 君为我们提供了两个创建`CFRunLoopObserver`的接口：

`CFRunLoopObserverCreate`以及`CFRunLoopObserverCreateWithHandler`。

两者的区别仅在于前者以函数指针的形式提供监听回调，后者以 block 的形式提供回调接口。

以`CFRunLoopObserverCreateWithHandler`为例，Observer 的实现大致如下：
![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-23-054507.jpg)

我们可以选择要监听的 runloop 事件，上面代码中使用了`kCFRunLoopAllActivities`，其他还有：`kCFRunLoopEntry`、`kCFRunLoopBeforeTimers`、`kCFRunLoopBeforeSources`、`kCFRunLoopBeforeWaiting`、`kCFRunLoopAfterWaiting`以及`kCFRunLoopExit`。

## Runloop Observer of Main Runloop

首先，我们通过 runloop observer 监听一下 main runloop。
![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-23-054520.png)

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-23-054530.png)

从上面的 log 我们可以得出：**main runloop 在主界面初始化完成，即将显示到屏幕前自动启动，runloop 启动后 (唤醒后) 会依次处理 timer(如果有)、source event(如果有)并在此前通知 observer。**

为了 log 显示简洁突出主题，下面我们只监听`kCFRunLoopEntry`、`kCFRunLoopBeforeWaiting`、`kCFRunLoopAfterWaiting`以及`kCFRunLoopExit`事件。

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-23-054545.jpg)

**main runloop 每分钟会被唤醒一次！(不明所以)**

在界面添加按钮，在其点击 handler 中 reload tableview：

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-23-054559.png)



![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-23-054613.png)

**UI 事件唤醒 main runloop 直到处理完该事件，main runloop 再次进入休眠。**

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-23-054632.png)

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-23-054639.png)]

**如果唤醒 runloop 的事件含有异步操作，runloop 不会等待异步操作完成。`viewWillAppear:`和`viewDidAppear:`不在同一次 runloop 中被调用。**

## Runloop Observer of Other Runloop

main runloop 作为整个 app 的神经中枢，很大程度上受系统所控制，下面我们通过 runloop observer 观察一下子线程的 runloop。

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-23-054707.png)



![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-23-054712.png)



![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-23-054728.png)



**timer 会唤醒 runloop 但不会使 runloop 退出**

如果子线程的 runloop 不添加任何 timer、source event：
![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-23-054748.png)



![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-23-054803.png)



**可以看到，此情况下，runloop 直接退出，连 runloop observer 都不通知一下 (runloop 根本没有启动)！**

下面再看一个更加复杂点的情况，添加第二个按钮，在其点击事件中向子线程发送 performSelector 消息：
![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-23-054904.png)



子线程的 timer handler、performSelector handler 如下：
![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-23-054914.png)



在子线程启动并触发 timer，但 timer handler 未返回之前，点击第二个 button：
![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-23-054941.png)



可以看到，子线程在 timer handler 返回后才处理 performSelector 消息，并且在一次 runloop 中可以处理多个事件。

奇怪的是，runloop 在处理完 performSelector 消息后，还处理了另一个 timer fire 事件，随后退出。

ok，小结一下：

*   main runloop 在主界面即将显示前由系统启动 (主界面 controller 的 `viewWillAppear:`执行后启动)；
*   runloop 启动后 (唤醒后) 会依次处理 timer(如果有)、source event(如果有)并在此前通知 observer；
*   main runloop 每分钟会被唤醒一次！(不明所以)；
*   UI 事件唤醒 main runloop 直到处理完该事件，如果该事件含有异步操作，runloop 不会等待异步操作完成；
*   UIViewController 的`viewWillAppear:`和`viewDidAppear:`不在同一次 runloop 中被调用；
*   timer 会唤醒 runloop 但不会使 runloop 退出；
*   如果子线程的 runloop 没有绑定 timer 或 source event，其 runloop 不会启动；
*   一次 runloop 可以处理多个事件。

# Autorelease and Runloop

* * *

在 MRC 中，autorelase 作为重要的内存管理机制而被广大程序猿们烂记于心。我们都知道，autorelease 对象会被放入 autorelease pool，最终会被自动 release！借用一句广告词：**有了 autorelease，程序猿们再也不用担心内存泄漏了！**

那么，问题来了：autorelease pool 中的对象到底什么时候会被真正的 release？

嗯，这也是一道很好的面试题！

答案也很简单，每次 runloop 退出前都会处理 autorelease pool——将其中的所有 object 都 release 一次。

咱们还是通过代码来看看吧，将前面提到的子线程的 performSelector 实现改成如下所示：
![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-23-055000.png)



![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-23-055013.png)

是的，如预期所料，`testAutoreleaseObj`作为 autorelease object 在 runloop 退出前被 release 了。

autorelease 固然好用，但在使用过程中也需谨慎，否则容易出问题，并且由 autorelease 引发的问题一般较难排查。
常见问题有：

*   autorelease object 被手动 release；
*   跨 runloop 使用 autorelease object。

一旦在大型项目中出现第一个问题，很难排查，其 crash 堆栈如下：
![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-23-055025.png)

从这个堆栈中我们能得到的只有：autorelease object 提前被手动 release！
呵呵，至于是哪个 object 被提前 release 了，不得而知！

第二个问题主要出现在类的对象含有 autorelease 的成员变量：
![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-23-055044.png)



`testViewController`类的实例保存了 autorelease 的`_str`，由于`viewDidAppear:`的调用在另一 runloop 中，此时`_str`已经释放！

是的，在类的成员变量中保存 autorelease 对象是一件十分危险的事情！

# 总结

* * *

runloop 作为事件处理的神经中枢，对整个 app 的意义不言而喻。准确而深入地了解 runloop 的机制，对程序猿的意义也不言而喻！

这两篇文章，我们简要介绍了 runloop 的基本概念、通过 pseudo-code 遐想了 runloop 的实现机制、通过 runloop observer 偷窥了 runloop 的重要时刻。

当然，runloop 还有很多问题值得探索…