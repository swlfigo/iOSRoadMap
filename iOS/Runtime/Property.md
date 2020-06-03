# iOS 属性

### 1.成员变量 实例变量 属性

![image-20190313113006579](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-19-014905.png)

图中的**Member Variable declarations**翻译过来就是**成员变量的声明**

```objective-c
类： Class (description/template for an object)
实例： Instance (manifestation of a class)
消息： Message (sent to object to make it act)
方法： Method (code invoked by a Message)
实例变量： Instance Variable (object-specific storage)
超类/子类： Superclass/Subclass (Inheritance)
协议：  Protocol (non-class-specific methods)
```

从给出的英文说明，可以看出：实例（Instance）是针对 类（class）而言的。实例是指类的声明；由此推理，实例变量（Instance Variable） 是指**由类声明的对象**。

 严格说来，上图中的  int  count;  是一个成员变量。而 `NSString name；` 是一个实例变量（NSString是一个类).至于 id data 应该属于成员变量还是实例变量呢？  因为 id 是 OC特有的类型。从本质上讲， id 等同于 （void *）。 所以 id data 应属于 实例变量。

**成员变量**：通常是指向对象或是基础类型（int, float）的简单指针。可以在.h 或是 .m 文件中声明：

**实例变量**：是成员变量的一种，实例是针对类而言的，是指对类的声明；由此推理，实例变量是指由类声明的对象。



### 2.  @synthesizer

@synthesize 语句只能被用在 @implementation 代码段中，@synthesize的作用就是让编译器为你自动生成setter与getter方法，@synthesize 还有一个作用，可以指定与属性对应的实例变量，例如@synthesize myButton = xxx；那么self.myButton其实是操作的实例变量xxx，而不是_myButton了。

如果.m文件中写了@synthesize myButton;那么生成的实例变量就是myButton；如果没写@synthesize myButton;那么生成的实例变量就是_myButton。



### 3.  @property

Objective-C2.0中的新语法：**Properties**。**它帮我们自动生成getter和setter**

写`@property`声明属性，其实是做了三件事

- .h: 声明了getter和setter方法；
- .h: 声明了实例变量(默认:下划线+属性名)；
- .m: 实现了getter和setter方法。

 **@property = Ivar + getter + setter**



### 4.  property 关键字

读写权限

- readonly
- readwrite      √默认关键字

引用计数

- retain / strong

  都是强引用，除了某些情况下不一样，比如修饰block，其他的时候也是可以通用的。

  (**external** 为 Block 外属性)

  ![](https://sylarimage.oss-cn-shenzhen.aliyuncs.com/MRC、ARCBlock关键字.png)

- weak / assign

  assign:

  修饰基本数据类型，如int, bool等

  修饰对象类型时，不改变其引用计数

  会产生悬垂指针：仍然指向内存地址，如果没覆盖后还调动变量就会crash

  weak：

  不改变修饰对象的引用计数

  所指对象在释放之后会自动设置为nil

- copy

| name       | 浅拷贝 | 深拷贝 |
| ---------- | ------ | ------ |
| 新内存空间 | 不分配 | 分配   |
| 引用计数   | 影响   | 不影响 |



| 源对象类型    | 拷贝方式    | 目标对象类型 | 拷贝类型 |
| ------------- | ----------- | ------------ | -------- |
| mutable对象   | copy        | 不可变       | 深拷贝   |
| mutable对象   | mutableCopy | 可变         | 深拷贝   |
| immutable对象 | copy        | 不可变       | 浅拷贝   |
| immutable对象 | mutableCopy | 可变         | 深拷贝   |

原子性

- atomic     √默认关键字
- nonatomic

atomic` 保证赋值获取是线程安全,是对成员属性的直接的获取安全，并不代表操作和访问安全.`

**atomic是自旋锁，即当上一线程没有执行完毕（被锁住），下一线程会一直等待（不会进入睡眠状态），当上一线程执行完毕，下一线程立即执行。他区别于互斥锁，互斥锁在等待的时候，会进入睡眠状态，当上一个线程执行完毕，睡眠状态就会被唤醒，然后再执行。**

比如 `atomic` 修饰的是一个数组,对数组**赋值获取**是安全的，但是对数组**进行操作**(添加对象，移除对象)是不保证线程不安全的.而且采用`atomic`消耗比较大

```objective-c
array = [[NSArray alloc]init];	//安全
[array addobject:obj];	//也会存在不安全
```



补充介绍 **`weak`关键字**：

实现原理 `weak`修饰时，`runtime`会维护一个`hash`表（也称为`weak`表）,用于存储对象的所有`weak`指针，`hash`表的`key`是该对象的地址，`value`为`weak`指针的地址（这个地址的值是所指对象的地址）**数组**。（备注`strong`是通过`runtime`维护的一个自动引用计数表） 

`weak`的实现原理总结：

1. 初始化时，`runtime`会调用`objc_initWeak`函数，初始化一个新的`weak`指针指向对象地址；
2. 添加引用时，`objc_initWeak`函数会调用`objc_storeWeak`函数，`objc_storeWeak`的作用是更新指针指向，创建对应的弱引用表（hash表)
3. 释放时，调用`clearDeallocating`函数。`clearDeallocating`函数首先根据对象地址获取`weak`指针地址的数组，然后遍历这个数组把其中指向空对象的指针设为`nil`，最后把这个指针从`weak`表中删除,最后清理对象的记录。



关于ARC下，不显示指定属性关键字时，默认关键字：
1.基本数据类型：atomic readwrite assign
2.普通OC对象： atomic readwrite strong

