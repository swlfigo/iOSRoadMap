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

 @property = Ivar + getter + setter