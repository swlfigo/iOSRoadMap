#  16. Runloop

Apple 官方给 `Runloop` 的定义：Runloop 是线程的基础支撑，是循环处理事件的机制，一个具体的 Runloop 就是一个事件处理循环。

**Runloop 的目的是使线程在没有事情可做时进入休眠状态，避免 CPU 空转。**

> Run loops are part of the fundamental infrastructure associated with threads. A run loop is an event processing loop that you use to schedule work and coordinate the receipt of incoming events. The purpose of a run loop is to keep your thread busy when there is work to do and put your thread to sleep when there is none.

## Runloop Modes

`Runloop Mode` 是`事件源的集合` + `Runloop`观察者的集合。**Runloop 每次都运行在某个特定的 mode 上。**

之所以要引入 mode 的概念，是**希望 Runloop 在监听过程中过滤掉不关心的事件源，只专注于某些特定的事件。**

![](http://img.isylar.com/media/15441532587740.png)

如果某个 `input source` 所属的 mode 不是当前监听的 mode，那 **其产生的所有事件都将被 hold 住，直到 runloop 运行在与其匹配的 mode 上。**

注意：当有 UI 滑动事件时，系统会将`Main Runloop`切换到`NSEventTrackingRunLoopMode`，以限定此时的事件源，确保滑动的流畅性。

同时，在默认情况下`NSRunLoopCommonModes`包含`NSEventTrackingRunLoopMode`，也就是说与`NSRunLoopCommonModes`关联的事件源也与`NSEventTrackingRunLoopMode`关联。

而此时，如果有子线程想通过`performSelecorOnMainThread...` 或 `dispatch_async(dispatch_get_main_queue(),^{})` 在主线程上执行某 selector，默认情况下上述两种方式产生的事件是关联到`NSRunLoopCommonModes`，因此在 UI 滑动时也会响应该事件并执行指定的 selector，从而影响滑动的流畅性。

为了避免此问题，可以封装上述接口，使其指定的 selector 运行在 Main Runloop 的其他 mode 上，如：`NSDefaultRunLoopMode`。

## Runloop & Thread

每个 `thread` 都有自己的 `Runloop`，可以通过 `NSRunLoop`的类方法`currentRunLoop`获取**当前线程**的 `runloop`。但只有 `main thread` 的 `runloop` **默认是开启**的，**其他线程如果希望持续存活下去，就需要手动开启runloop**。

## Summary

* main runloop在主界面即将显示前由系统启动(主界面 controller 的 viewWillAppear:执行后启动)；

* runloop 启动后(唤醒后)会依次处理 timer(如果有)、source event(如果有)并在此前通知 observer；

* main runloop每分钟会被唤醒一次

* UI事件唤醒 main runloop 直到处理完该事件，如果该事件含有异步操作，runloop 不会等待异步操作完成；

* UIViewController的viewWillAppear:和viewDidAppear:不在同一次 runloop 中被调用；

* timer会唤醒 runloop 但不会使 runloop 退出；

* 如果子线程的 runloop 没有绑定 timer 或 source event，其 runloop 不会启动；

* 一次 runloop 可以处理多个事件。

## Reference

[1.谜一样的 Runloop](http://zxfcumtcs.github.io/2014/11/15/runloop/)

