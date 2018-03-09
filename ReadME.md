
#### 更多博文可看传送门 ↓

[传送门](http://isylar.com)


## iOS

### 1. 简述下 UIViewController 的生命周期

ViewController的生命周期中各方法扫行须序如下：

alloc init—>

loadView—> 尽管不直接调用该方法，如多手动创建自己的视图，那么应该覆盖这个方法并将它们赋值给试图控制器的 view 属性。

viewDidLoad—> 只有在视图控制器将其视图载入到内存之后才调用该方法，这是执行任何其他初始化操作的入口。

viewWillAppear—> 当试图将要添加到窗口中并且还不可见的时候或者上层视图移出图层后本视图变成顶级视图时调用该方法，用于执行诸如改变视图方向等的操作。实现该方法时确保调用 [super viewWillAppear:]

viewDidAppear—> 当视图添加到窗口中以后或者上层视图移出图层后本视图变成顶级视图时调用，用于放置那些需要在视图显示后执行的代码。确保调用 [super viewDidAppear:]

viewWillDisappear—>
viewDidDisappear—>
viewWillUnload->
viewDidUnload—>
dealloc->

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

### 3. runloop与线程有什么关系?

1. runloop与线程是一一对应的，一个runloop对应一个核心的线程，为什么说是核心的，是因为runloop是可以嵌套的，但是核心的只能有一个，他们的关系保存在一个全局的字典里。
2. runloop是来管理线程的，当线程的runloop被开启后，线程会在执行完任务后进入休眠状态，有了任务就会被唤醒去执行任务。
3. runloop在第一次获取时被创建，在线程结束时被销毁。
4. 对于主线程来说，runloop在程序一启动就默认创建好了。
5. 对于子线程来说，runloop是懒加载的，只有当我们使用的时候才会创建，所以在子线程用定时器要注意：确保子线程的runloop被创建，不然定时器不会回调。

### 4. Autoreleasepool 有什么作用 , 如何实现 , 什么时候释放

AutoreleasePool（自动释放池）是OC中的一种内存自动回收机制，它可以延迟加入AutoreleasePool中的变量release的时机。

在没有手加Autorelease Pool的情况下，Autorelease对象是在当前的runloop迭代结束时释放的，而它能够释放的原因是系统在每个runloop迭代中都加入了自动释放池Push和Pop

AutoreleasePool的内存结构就是一个双向链表。一个线程的autoreleasepool就是一个指针栈。
栈中存放的指针指向加入需要release的对象或者POOL_SENTINEL（哨兵对象，用于分隔autoreleasepool）。
栈中指向POOL_SENTINEL的指针就是autoreleasepool的一个标记。当autoreleasepool进行出栈操作，每一个比这个哨兵对象后进栈的对象都会release。
这个栈是由一个以page为节点双向链表组成，page根据需求进行增减。
autoreleasepool对应的线程存储了指向最新page（也就是最新添加autorelease对象的page）的指针。


