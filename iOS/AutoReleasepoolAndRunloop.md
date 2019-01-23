#  Runloop & Autoreleasepool & ARC

个人认为将 `AutoReleasepool` 、 `ARC` 、 `Runloop` 3种技术联系在一起来看是一种全面的认知.


## Runloop 与 AutoReleasepool 创建
 每个`Runloop`中都会创建一个 `AutoReleasepool` 并在 `Runloop迭代结束`进行释放。何为 `迭代结束`？当前`Runloop` 进入 `Sleep mode`的时候,就结束当前 `Runloop`迭代.新的一轮`Runloop`创建一个新的 `AutoReleasepool`, `Pool`里面的临时对象在结束后得到释放(不一定即时,也有可能延后,系统决定)
 
 `Runloop`第一个 Observer 监视的事件是 Entry(即将进入Loop)，其回调内会调用 `_objc_autoreleasePoolPush()` 创建自动释放池。其 order 是-2147483647，优先级最高，保证创建释放池发生在其他所有回调之前。

 第二个 Observer 监视了两个事件： BeforeWaiting(准备进入休眠) 时调用`_objc_autoreleasePoolPop()` 和 `_objc_autoreleasePoolPush()` 释放旧的池并创建新池；Exit(即将退出Loop) 时调用 `_objc_autoreleasePoolPop()` 来释放自动释放池。这个 Observer 的 order 是 2147483647，优先级最低，保证其释放池子发生在其他所有回调之后。



## Runloop 流程
 `Runloop`具体流程可以参考下文 1,2.此处添加原文图片引用,详细内容可参考原文

![](http://img.isylar.com/media/15482151348336.jpg)

![rl00](http://img.isylar.com/media/rl00.png)



##  Runloop Mode
 `Runloop`总是运行在某种特定的CFRunLoopModeRef下,意思是每次`Runloop`开始时候会选择一个mode，执行这个mode里面的 `block`,`timer`等事件.这可以解释滑动过程中，`NSTimer`为什么会停止,因为滑动过程中`Runloop`处于 `TrackingMode`,`NSTimer`默认添加在`DefaultMode`,所以不执行


## Runloop 运用场景
 
`AFNetworking2.x` 保活原理:

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

## `AutoReleasepoolPage`
ARC下，我们使用@autoreleasepool{}来使用一个AutoreleasePool，随后编译器将其改写成下面的样子：

```
void *context = objc_autoreleasePoolPush();
// {}中的代码
objc_autoreleasePoolPop(context);
```

而这两个函数都是对`AutoreleasePoolPage`的简单封装，所以自动释放机制的核心就在于这个类。

`AutoreleasePoolPage`是一个C++实现的类

![](http://img.isylar.com/media/15482230177420.jpg)


* AutoreleasePool并没有单独的结构，而是由若干个AutoreleasePoolPage以双向链表的形式组合而成（分别对应结构中的parent指针和child指针）
* AutoreleasePool是按线程一一对应的（结构中的thread指针指向当前线程）
* AutoreleasePoolPage每个对象会开辟4096字节内存（也就是虚拟内存一页的大小），除了上面的实例变量所占空间，剩下的空间全部用来储存autorelease对象的地址
* 上面的id *next指针作为游标指向栈顶最新add进来的autorelease对象的下一个位置
* 一个AutoreleasePoolPage的空间被占满时，会新建一个AutoreleasePoolPage对象，连接链表，后来的autorelease对象在新的page加入

当前线程中只有一个AutoreleasePoolPage对象，并记录了很多autorelease对象地址时内存如下图：
![](http://img.isylar.com/media/15482231312850.jpg)

释放:

每当进行一次`objc_autoreleasePoolPush`调用时，runtime向当前的AutoreleasePoolPage中add进一个哨兵对象，值为0（也就是个nil），那么这一个page就变成了下面的样子：

![](http://img.isylar.com/media/15482231833235.jpg)

`objc_autoreleasePoolPush`的返回值正是这个哨兵对象的地址，被`objc_autoreleasePoolPop`(哨兵对象)作为入参，于是：

* 根据传入的哨兵对象地址找到哨兵对象所处的page
* 在当前page中，将晚于哨兵对象插入的所有autorelease对象都发送一次- release消息，并向回移动next指针到正确位置
* 从最新加入的对象`一直向前清理`，可以向前跨越若干个page，直到哨兵所在的page

刚才的objc_autoreleasePoolPop执行后，最终变成了下面的样子：
![](http://img.isylar.com/media/15482232798459.jpg)


## 手动@autoreleasepool 与 嵌套

嵌套`autorelesepool`很好解释,pop的时候总会释放到上次push的位置为止，多层的pool就是多个哨兵对象而已，就像剥洋葱一样，每次一层，互不影响。

手动`autoreleasepool`,如下文参考2例子,可以得知这个`for`循环中，每一次循环会清理掉一次内存,因为完全执行完 `for`循环才会，`runloop`才会进行休眠，如果说是按照系统的`autoreleasepool`来说，应该是休眠前才释放，但是，文中demo内存并没有显示出循环中内存暴涨，这也说明了，**手动autorelesepool 不是在内存峰值时候释放**

## 线程与runloop
线程和 RunLoop 之间是一一对应的，其关系是保存在一个全局的 Dictionary 里。线程刚创建时并没有 RunLoop，如果你不主动获取，那它一直都不会有。RunLoop 的创建是发生在第一次获取时，RunLoop 的销毁是发生在线程结束时。你只能在一个线程的内部获取其 RunLoop（主线程除外）。


## Reference

[1. 解密Runloop](http://mrpeak.cn/blog/ios-runloop/)

[2. 在ARC环境中autoreleasepool(runloop)的研究](https://juejin.im/post/59eabe2451882578ca2dc145)

[3. 黑幕背后的Autorelease](https://blog.sunnyxx.com/2014/10/15/behind-autorelease/)

[4. iOS RunLoop详解](https://juejin.im/post/5aca2b0a6fb9a028d700e1f8)

[5.深入了解Runloop](https://blog.ibireme.com/2015/05/18/runloop/)