# iOS面试题总结归纳,不定期更新
#### 总结面试过程中碰到问题

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



## 架构方面

### 1. 设计模式

#### MVC
 
##### 什么是MVC？

MVC最早存在于桌面程序中的, M是指业务数据, V是指用户界面, C则是控制器. 在具体的业务场景中, C作为M和V之间的连接, 负责获取输入的业务数据, 然后将处理后的数据输出到界面上做相应展示, 另外, 在数据有所更新时, C还需要及时提交相应更新到界面展示. 在上述过程中, 因为M和V之间是完全隔离的, 所以在业务场景切换时, 通常只需要替换相应的C, 复用已有的M和V便可快速搭建新的业务场景. MVC因其复用性, 大大提高了开发效率, 现已被广泛应用在各端开发中。


## 网络
### 1.简述TCP的三次握手过程

在TCP/IP协议中,TCP协议提供可靠的连接服务,采用三次握手建立一个连接.
1）第一次握手：
Client将标志位SYN置为1，随机产生一个值seq=J，并将该数据包发送给Server，Client进入SYN_SENT状态，等待Server确认。

（2）第二次握手：
Server收到数据包后由标志位SYN=1知道Client请求建立连接，Server将标志位SYN和ACK都置为1，ack=J+1，随机产生一个值seq=K，并将该数据包发送给Client以确认连接请求，Server进入SYN_RCVD状态。

（3）第三次握手：
Client收到确认后，检查ack是否为J+1，ACK是否为1，如果正确则将标志位ACK置为1，ack=K+1，并将该数据包发送给Server，Server检查ack是否为K+1，ACK是否为1，如果正确则连接建立成功，Client和Server进入ESTABLISHED状态，完成三次握手，随后Client与Server之间可以开始传输数据了。


### 2. 4次挥手过程详解

 * 第一次挥手：
Client发送一个FIN，用来关闭Client到Server的数据传送，Client进入FIN_WAIT_1状态。
 * 第二次挥手：
Server收到FIN后，发送一个ACK给Client，确认序号为收到序号+1（与SYN相同，一个FIN占用一个序号），Server进入CLOSE_WAIT状态。
 * 第三次挥手：
Server发送一个FIN，用来关闭Server到Client的数据传送，Server进入LAST_ACK状态。
 * 第四次挥手：
Client收到FIN后，Client进入TIME_WAIT状态，接着发送一个ACK给Server，确认序号为收到序号+1，Server进入CLOSED状态，完成四次挥手。


### 3. TCP/UDP区别以及UDP如何实现可靠传输

TCP和UDP是OSI模型中的运输层中的协议。TCP提供可靠的通信传输，而UDP则常被用于让广播和细节控制交给应用的通信传输。

**TCP与UDP区别总结：**

1、TCP面向连接（如打电话要先拨号建立连接）;UDP是无连接的，即发送数据之前不需要建立连接

2、TCP提供可靠的服务。也就是说，通过TCP连接传送的数据，无差错，不丢失，不重复，且按序到达;UDP尽最大努力交付，即不保证可靠交付
3、TCP面向字节流，实际上是TCP把数据看成一连串无结构的字节流;UDP是面向报文的
UDP没有拥塞控制，因此网络出现拥塞不会使源主机的发送速率降低（对实时应用很有用，如IP电话，实时视频会议等）
4、每一条TCP连接只能是点到点的;UDP支持一对一，一对多，多对一和多对多的交互通信
5、TCP首部开销20字节;UDP的首部开销小，只有8个字节
6、TCP的逻辑通信信道是全双工的可靠信道，UDP则是不可靠信道

**UDP如何实现可靠传输**

由于在传输层UDP已经是不可靠的连接，那就要在应用层自己实现一些保障可靠传输的机制

简单来讲，要使用UDP来构建可靠的面向连接的数据传输，就要实现类似于TCP协议的

超时重传（定时器）

有序接受 （添加包序号）

应答确认 （Seq/Ack应答机制）

滑动窗口流量控制等机制 （滑动窗口协议）

等于说要在传输层的上一层（或者直接在应用层）实现TCP协议的可靠数据传输机制，比如使用UDP数据包+序列号，UDP数据包+时间戳等方法。

### 4. Http 和 Https 有什么关系和区别

HTTP协议传输的数据都是未加密的，也就是明文的，因此使用HTTP协议传输隐私信息非常不安全，为了保证这些隐私数据能加密传输，于是网景公司设计了SSL（Secure Sockets Layer）协议用于对HTTP协议传输的数据进行加密

HTTPS和HTTP的区别主要如下：

　　1、https协议需要到ca申请证书，一般免费证书较少，因而需要一定费用。

　　2、http是超文本传输协议，信息是明文传输，https则是具有安全性的ssl加密传输协议。

　　3、http和https使用的是完全不同的连接方式，用的端口也不一样，前者是80，后者是443。

　　4、http的连接很简单，是无状态的；HTTPS协议是由SSL+HTTP协议构建的可进行加密传输、身份认证的网络协议，比http协议安全。

### 5. get 和 post 区别

//简略回答就好

* GET后退按钮/刷新无害，POST数据会被重新提交（浏览器应该告知用户数据会被重新提交）。
* GET书签可收藏，POST为书签不可收藏。
* GET能被缓存，POST不能缓存 
* GET对数据长度有限制，当发送数据时，GET 方法向 URL 添加数据；URL 的长度是受限制的（URL 的最大长度是 2048 个字符）。POST无限制。
* GET只允许 ASCII 字符。POST没有限制。也允许二进制数据。与 POST 相比，GET 的安全性较差，因为所发送的数据是 URL 的一部分。在发送密码或其他敏感信息时绝不要使用 GET ！POST 比 GET 更安全，因为参数不会被保存在浏览器历史或 web 服务器日志中。
* GET的数据在 URL 中对所有人都是可见的。POST的数据不会显示在 URL 中。




## 一些推荐阅读

1. [《图解HTTP》知识点摘录](https://juejin.im/post/5aa62f93f265da23906ba830)
2. 

