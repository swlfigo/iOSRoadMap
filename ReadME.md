# iOS面试题总结归纳,不定期更新
#### 总结面试过程中碰到问题

#### 更多博文可看传送门 ↓

[传送门](http://isylar.com)


## iOS

[简述下 UIViewController 的生命周期](/iOS/UIViewController 的生命周期.md)

### 2. OC中向一个nil对象发送消息将会发生什么?为什么?

**关于nil**
nil的定义是null pointer to object-c object，指的是一个OC对象指针为空，本质就是(id)0，是OC对象的字面0值

不过这里有必要提一点就是OC中给空指针发消息不会崩溃的语言特性，**原因是OC的函数调用都是通过objc_msgSend进行消息发送来实现的**，相对于C和C++来说，对于空指针的操作会引起Crash的问题，**而objc_msgSend会通过判断self来决定是否发送消息，如果self为nil，那么selector也会为空，直接返回**，所以不会出现问题。

**课外知识**

```objective-c
// runtime.h（类在runtime中的定义）
// http://weibo.com/luohanchenyilong/
// https://github.com/ChenYilong

struct objc_class {
    Class isa OBJC_ISA_AVAILABILITY; //isa指针指向Meta Class，因为Objc的类的本身也是一个Object，为了处理这个关系，runtime就创造了Meta Class，当给类发送[NSObject alloc]这样消息时，实际上是把这个消息发给了Class Object
#if !__OBJC2__
    Class super_class OBJC2_UNAVAILABLE; // 父类
    const char *name OBJC2_UNAVAILABLE; // 类名
    long version OBJC2_UNAVAILABLE; // 类的版本信息，默认为0
    long info OBJC2_UNAVAILABLE; // 类信息，供运行期使用的一些位标识
    long instance_size OBJC2_UNAVAILABLE; // 该类的实例变量大小
    struct objc_ivar_list *ivars OBJC2_UNAVAILABLE; // 该类的成员变量链表
    struct objc_method_list **methodLists OBJC2_UNAVAILABLE; // 方法定义的链表
    struct objc_cache *cache OBJC2_UNAVAILABLE; // 方法缓存，对象接到一个消息会根据isa指针查找消息对象，这时会在method Lists中遍历，如果cache了，常用的方法调用时就能够提高调用的效率。
    struct objc_protocol_list *protocols OBJC2_UNAVAILABLE; // 协议链表
#endif
} OBJC2_UNAVAILABLE;

```

objc在向一个对象发送消息时，runtime库会根据对象的isa指针找到该对象实际所属的类，然后在该类中的方法列表以及其父类方法列表中寻找方法运行，然后在发送消息的时候，objc_msgSend方法不会返回值，所谓的返回内容都是具体调用时执行的。那么，回到本题，如果向一个nil对象发送消息，首先在寻找对象的isa指针时就是0地址返回了，所以不会出现任何错误。

### 3. Runloop与线程有什么关系?

1. runloop与线程是一一对应的，一个runloop对应一个核心的线程，为什么说是核心的，是因为runloop是可以嵌套的，但是核心的只能有一个，他们的关系保存在一个全局的字典里。
2. runloop是来管理线程的，当线程的runloop被开启后，线程会在执行完任务后进入休眠状态，有了任务就会被唤醒去执行任务。
3. runloop在第一次获取时被创建，在线程结束时被销毁。
4. 对于主线程来说，runloop在程序一启动就默认创建好了。
5. 对于子线程来说，runloop是懒加载的，只有当我们使用的时候才会创建，所以在子线程用定时器要注意：确保子线程的runloop被创建，不然定时器不会回调。

### 4. Autoreleasepool 有什么作用 , 如何实现 , 什么时候释放

AutoreleasePool（自动释放池）是OC中的一种内存自动回收机制，它可以延迟加入AutoreleasePool中的变量release的时机。

在没有手加Autorelease Pool的情况下， **Autorelease对象是在当前的runloop迭代结束时释放的，而它能够释放的原因是系统在每个runloop迭代中都加入了自动释放池Push和Pop**

AutoreleasePool的内存结构就是一个双向链表。一个线程的autoreleasepool就是一个指针栈。
栈中存放的指针指向加入需要release的对象或者POOL_SENTINEL（哨兵对象，用于分隔autoreleasepool）。
栈中指向POOL_SENTINEL的指针就是autoreleasepool的一个标记。当autoreleasepool进行出栈操作，每一个比这个哨兵对象后进栈的对象都会release。
这个栈是由一个以page为节点双向链表组成，page根据需求进行增减。
autoreleasepool对应的线程存储了指向最新page（也就是最新添加autorelease对象的page）的指针。

### 5.KVO内部实现原理

* KVO是基于runtime机制实现的
* 当某个类的属性对象`第一次被观察`时，系统就会在运行期动态地创建该类的一个派生类，在这个派生类中重写基类中任何被观察属性的setter 方法。派生类在被重写的setter方法内实现真正的`通知机制`
* 如果原类为Person，那么生成的派生类名为NSKVONotifying_Person
* 每个类对象中都有一个isa指针指向当前类，当一个类对象的第一次被观察，那么系统会偷偷将isa指针指向动态生成的派生类，从而在给被监控属性赋值时执行的是派生类的setter方法
* 键值观察通知依赖于NSObject 的两个方法: `willChangeValueForKey:` 和 `didChangevlueForKey:` 在一个被观察属性发生改变之前， `willChangeValueForKey: `一定会被调用，这就会记录旧的值。而当改变发生后，`didChangeValueForKey:` 会被调用，继而 `observeValueForKey:ofObject:change:context: `也会被调用。
补充：KVO的这套实现机制中苹果还偷偷重写了class方法，让我们误认为还是使用的当前类，从而达到隐藏生成的派生类

![](http://okslxr2o0.bkt.clouddn.com/15213019185506.jpg)

### 6.Runtime 是如何找到实例方法的具体实现的？

一个OC方法被编译成objc_msgSend，因为OC中存在一种对象叫做类对象（Class Object），类对象中保存了方法的列表，superClass等信息。objc_msgSend这个C函数输入参数包括，id类型的self，SEL类型的_cmd,以及参数列表。很直观，id类型中一定存在一个可以找到类对象的指针。 

* OC的对象通过isa找到类对象
* 类对象查找自己存储的方法列表来找到对应的方法执行体
* 方法执行体执行具体的代码，并返回值给调用者。

我们来看一个例子， 
定义一个

```objectivec
@interface CustomObject : NSObject
-(NSString *)returnMeHelloWorld;
@end
@implementation CustomObject

-(NSString *)returnMeHelloWorld{
    return @"hello world";
}
@end
```

我们先只看调用这一行

```objectivec
    NSString * helloWorld =  [obj returnMeHelloWorld];
```

* 编译成如下id objc_msgSend(self,@selector(returnMeHelloWorld:));
* 在self中沿着isa找到CustomObject的类对象
* 类对象查找自己的方法list，找到对应的方法执行体Method
* 把参数传递给IMP实际指向的执行代码
* 代码执行返回结果给helloWorld

### 7 __weak 实现原理的概括
Runtime维护了一个weak表，用于存储指向某个对象的所有weak指针。weak表其实是一个hash（哈希）表，Key是所指对象的地址，Value是weak指针的地址（这个地址的值是所指对象的地址）数组。

weak 的实现原理可以概括一下三步：

1. 初始化时：runtime会调用objc_initWeak函数，初始化一个新的weak指针指向对象的地址。

2. 添加引用时：objc_initWeak函数会调用 objc_storeWeak() 函数， objc_storeWeak() 的作用是更新指针指向，创建对应的弱引用表。

3. 释放时，调用clearDeallocating函数。clearDeallocating函数首先根据对象地址获取所有weak指针地址的数组，然后遍历这个数组把其中的数据设为nil，最后把这个entry从weak表中删除，最后清理对象的记录。

### 8. 离屏渲染

* 当设置某些UI图层属性时候，如果指定为被未预合成之前，不能直接显示在屏幕上的时候，就触发了离屏渲染。离屏渲染是基于GPU层面上的，指GPU在当前屏幕缓冲区外开辟了一个缓冲区，进行渲染操作。

**为何要避免离屏渲染?**

离屏渲染发生在GPU层面上，因为离屏渲染使GPU触发Opengl多通道渲染管线，产生额外开销，所以要避免。
在触发离屏渲染时候，会增加GPU工作量，增加GPU工作量，可能会导致GPU和CPU工作耗时的总耗时超出Vsync信号时间，导致UI卡顿或者掉帧。

### 9. 什么情况使用 weak 关键字，相比 assign 有什么不同？

在 ARC 中,在有可能出现循环引用的时候,往往要通过让其中一端使用 weak 来解决,比如: delegate 代理属性.

不同点:

1. weak 此特质表明该属性定义了一种“非拥有关系” (nonowning relationship)。为这种属性设置新值时，设置方法既不保留新值，也不释放旧值。此特质同assign类似， 然而在属性所指的对象遭到摧毁时，属性值也会清空(nil out)。 而 assign 的“设置方法”只会执行针对“纯量类型” (scalar type，例如 CGFloat 或 NSlnteger 等)的简单赋值操作。

2. assign 可以用非 OC 对象,而 weak 必须用于 OC 对象

### 10. Objective-C Associated Objects 的实现原理

1. 关联对象与被关联对象本身的存储并没有直接的关系，它是存储在单独的哈希表中的；
2. 关联对象的五种关联策略与属性的限定符非常类似，在绝大多数情况下，我们都会使用 **OBJC_ASSOCIATION_RETAIN_NONATOMIC** 的关联策略，这可以保证我们持有关联对象；
3. 关联对象的释放时机与移除时机并不总是一致，比如实验中用关联策略 **OBJC_ASSOCIATION_ASSIGN** 进行关联的对象，很早就已经被释放了，但是并没有被移除，而再使用这个关联对象时就会造成 Crash 。

### 11. iOS中图片的加载与渲染过程

 [详见子目录下md文件 - 传送门](https://github.com/swlfigo/iOSInterview/blob/master/杂乱知识点/iOSPicLoadAndRender.md)

## 线程

### 1. 浅谈GCD
##### 1. GCD的两个核心概念是:`任务`和`队列`. 

###### 任务与队列

* 任务 : 在block中执行的代码块 
* 队列 : 用来存放任务的
  
###### 队列 和 线程的区别: 
队列中存放的任务最后都要由线程来执行! 
队列的原则:先进先出,后进后出

###### 队列分类：
1.串行队列 2.并发队列 3.主队列 4.全局队列

* 串行队列:任务一个接一个的执行 
* 并发队列:队列中的任务并发执行 
* 主队列:跟主线程相关的队列,主队列里面的内容都会在主线程中执行 
* 全局队列:一个特殊的并发队列

###### 并发队列和全局队列的区别: 
* 并发队列有名称,可以跟踪错误.全局队列没有. 
* 在ARC中两个队列不需要考虑释放内存,但是在MRC中并发队列创造出来的需要 release 操作,而全局队列只有一个不需要. 
* 一般在开发过程中我们使用全局队列

##### 2.同步和异步: 
###### 同步异步
同步:只能在当前线程中执行任务,不具备开启新线程的能力 
异步:可以在新的线程中执行任务,具备开启新线程的能力

同步执行任务: 
dispatch_sync(队列,任务) 
异步执行: 
dispatch_async(队列,任务)

###### 队列和执行方式组合的效果:
1) 串行队列同步执行，既在当前线程中顺序执行 
2) 串行队列异步执行，开辟一条新的线程，在该线程中顺序执行 
3) 并行队列同步执行，不开辟线程，在当前线程中顺序执行 
4) 并行队列异步执行，开辟多个新的线程，并且线程会重用，无序执行 
5) 主队列异步执行，不开辟新的线程，顺序执行 
6) 主队列同步执行，会造成死锁（’主线程’和’主队列’相互等待,卡住主线程）

### 2. NSOprationQueue 与 GCD 的区别与选用

1. GCD是底层的C语言构成的API，而NSOperationQueue及相关对象是Objc的对象。在GCD中，在队列中执行的是由block构成的任务，这是一个轻量级的数据结构；而Operation作为一个对象，为我们提供了更多的选择；

2. 在NSOperationQueue中，我们可以随时取消已经设定要准备执行的任务(当然，已经开始的任务就无法阻止了)，而GCD没法停止已经加入queue的block(其实是有的，但需要许多复杂的代码)；

3. GCD 只支持FIFO 的队列，而NSOperationQueue可以调整队列的执行顺序（通过调整权重）。NSOperationQueue可以方便的管理并发、NSOperation之间的优先级。


4. GCD优点：GCD主要与block结合使用。


**引申:**

**使用NSOperation和NSOperationQueue的优点：**

1. 可以取消操作：在运行任务前，可以在NSOperation对象调用cancel方法，标明此任务不需要执行。但是GCD队列是无法取消的，因为它遵循“安排好之后就不管了（fire and forget）”的原则。
2. 可以指定操作间的依赖关系：例如从服务器下载并处理文件的动作可以用操作来表示。而在处理其他文件之前必须先下载“清单文件”。而后续的下载工作，都要依赖于先下载的清单文件这一操作。
3. 监控NSOperation对象的属性：可以通过KVO来监听NSOperation的属性：可以通过isCancelled属性来判断任务是否已取消；通过isFinished属性来判断任务是否已经完成。
4. 可以指定操作的优先级：操作的优先级表示此操作与队列中其他操作之间的优先关系，我们可以指定它

### 2. 线程与进程
**进程:** 在系统中正在运行的一个应用程序,每个进程之间都是独立的，每个进程均运行在其专用且受保护的内存空间内。进程是CPU分配资源和调度的单位。
**线程:** 一个进程(程序)的所有任务都在线程中执行，每个进程至少有一个线程(主线程)。线程是CPU调度（执行任务）的最小单位，其实质就是一段代码（一个任务）。

## 架构方面

### 1. 设计模式

#### MVC
 
##### 什么是MVC？

MVC最早存在于桌面程序中的, M是指业务数据, V是指用户界面, C则是控制器. 在具体的业务场景中, C作为M和V之间的连接, 负责获取输入的业务数据, 然后将处理后的数据输出到界面上做相应展示, 另外, 在数据有所更新时, C还需要及时提交相应更新到界面展示. 在上述过程中, 因为M和V之间是完全隔离的, 所以在业务场景切换时, 通常只需要替换相应的C, 复用已有的M和V便可快速搭建新的业务场景. MVC因其复用性, 大大提高了开发效率, 现已被广泛应用在各端开发中。


## 网络
### 1.简述TCP的三次握手过程 - [链接](https://github.com/swlfigo/iOSInterview/blob/master/网络/简述TCP的三次握手过程.md)

### 2. 4次挥手过程详解 - [链接](https://github.com/swlfigo/iOSInterview/blob/master/网络/4次挥手过程详解.md)

### 3. TCP/UDP区别以及UDP如何实现可靠传输 - [链接](https://github.com/swlfigo/iOSInterview/blob/master/网络/TCP:UDP区别以及UDP如何实现可靠传输.md)

### 4. Http 和 Https 有什么关系和区别 - [链接](https://github.com/swlfigo/iOSInterview/blob/master/网络/Http%20和%20Https%20有什么关系和区别.md)

### 5. get 和 post 区别 - [链接](https://github.com/swlfigo/iOSInterview/blob/master/网络/get%20和%20post%20区别.md)

### 6. 什么是Http协议无状态协议?怎么解决Http协议无状态协议? - [链接](https://github.com/swlfigo/iOSInterview/blob/master/网络/什么是Http协议无状态协议%3F怎么解决Http协议无状态协议%3F.md)

### 7. 一次完整的HTTP请求所经历的7个步骤 - [链接](https://github.com/swlfigo/iOSInterview/blob/master/网络/一次完整的HTTP请求所经历的7个步骤.md)

## 一些推荐阅读

1. [《图解HTTP》知识点摘录](https://juejin.im/post/5aa62f93f265da23906ba830)
2. [iOS 消息发送与转发详解](https://juejin.im/post/5aa79411f265da237a4cb045)




-------

## 杂乱知识点

 1.App启动过程 - [链接](https://github.com/swlfigo/iOSInterview/blob/master/%E6%9D%82%E4%B9%B1%E7%9F%A5%E8%AF%86%E7%82%B9/App%E5%90%AF%E5%8A%A8.md)

2.Cocoapods原理总结 - [链接](https://juejin.im/entry/59dd94b06fb9a0451463030b)

3.内联函数,与宏的区别 - [链接](https://github.com/swlfigo/iOSInterview/blob/master/%E6%9D%82%E4%B9%B1%E7%9F%A5%E8%AF%86%E7%82%B9/static_inline.md)

