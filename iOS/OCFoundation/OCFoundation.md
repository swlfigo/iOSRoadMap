# OC 语言特性

## OC 基础知识



###  1. 成员变量 实例变量 属性

![image-20190313113006579](./assets/image-20190313113006579.png)

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
 严格说来，上图中的  int  count;  是一个成员变量。
 而 **NSString name；*  是一个实例变量（NSString是一个类）。
 至于 id data 应该属于成员变量还是实例变量呢？  因为 id 是 OC特有的类型。从本质上讲， id 等同于 （void *）。 所以 id data 应属于 实例变量。

**成员变量**：通常是指向对象或是基础类型（int, float）的简单指针。可以在.h 或是 .m 文件中声明：

**实例变量**：是成员变量的一种，实例是针对类而言的，是指对类的声明；由此推理，实例变量是指由类声明的对象。

**属性**：GCC 到 LLVM（low level virtual machine），编译器自动为属性添加成员变量，规则：_属性名。如果需要自定义成员变量的名字，可以使用@synthesizer实现。