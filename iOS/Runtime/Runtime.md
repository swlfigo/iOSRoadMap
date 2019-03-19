# Runtime机制

Runtime内容流程

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-19-022544.jpg)

## Runtime数据结构

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-19-023218.jpg)

- superClass 指向当前类的父类
- cache_t 提供消息传递过程当中的缓存方法查找 ， 实质上是装满了 bucket_t 的一个 hash 表
- class_data_bits_t 类的基础信息，包含了类的方法列表，协议列表等。



## 类对象与元类对象

#### 1.metaclass & class

在objc中，class存储类的实例方法（-），meta class存储类的类方法（+），class的isa指针指向meta class。

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-19-023919.jpg)

一个对象（Instance of Subclass）的`isa`指针指向它所属的类 Subclass（class), Subclass（class）的`isa`指针指向 Subclass（meta），Subclass（meta）的 `isa`指针指向Root class(meta).Root class（meta）的 `isa`指针指向本身。
 同时，Root class（meta）的父类是Root class（class），即NSObject，NSObject的父类为nil。

每个类**仅有一个类对象**，而每个类对象**仅有一个与之相关的元类**。当你发出一个类似 `[NSObject alloc] `的消息时，你事实上是把这个消息发给了一个类对象 (Class Object) ，这个类对象必须是一个元类的实例，而这个元类同时也是一个根元类 (root meta class) 的实例。所有的元类最终都指向根元类为其超类。所有的元类的方法列表都有能够响应消息的类方法。所以当 [NSObject alloc] 这条消息发给类对象的时候，objc_msgSend() 会去它的元类里面去查找能够响应消息的方法，如果找到了，然后对这个类对象执行方法调用。





## Reference

[1.Objective-C Runtime机制简析(Article文件夹有收藏)](https://www.jianshu.com/p/0a4e5b944d7d)

[2.Objective-C Runtime](http://yulingtianxia.com/blog/2014/11/05/objective-c-runtime/)