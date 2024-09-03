

# iOS 对象与数据结构

不管是类对象还是元类对象，类型都是 Class，class 和 mete-class 的底层都是 objc_class 结构体的指针。

```objective-c
typedef struct objc_class *Class;
```



## OBJC2 结构

**objc_class**的真实定义实际的代码我们可以从 [objc-runtime-new.h](https://opensource.apple.com/source/objc4/objc4-750.1/runtime/objc-runtime-new.h.auto.html) 中看到(中间代码省略)：

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2020-04-23-143952.jpg)



**objc_object**的真实定义 详见 [objc-private.h文件](https://opensource.apple.com/source/objc4/objc4-750.1/runtime/objc-private.h.auto.html)

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2020-04-23-144015.jpg)

如上图,关系也如旧版本一样, **objc_class** 继承于 **objc_object**

objc_object中有一个isa指针，那么objc_class继承objc_object，也就同样拥有一个isa指针

所有继承自 `NSObject` 的类实例化后的对象都会包含一个类型为 `isa_t` 的结构体。

- `super_class` 指向当前类的父类
- `cache` 用于缓存指针和 `vtable`，加速方法的调用
- `bits` 就是存储类的方法、属性、遵循的协议等信息的地方



## class_rw_t

**objc_class** 中的 data 返回 `class_rw_t` 结构，此结构定义如下：



![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2020-04-23-144757.png)

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2020-04-23-144612.jpg)



而`class_rw_t`是通过bits调用data方法得来的，来到data方法内部实现。我们可以看到，data函数内部仅仅对bits进行`&FAST_DATA_MASK`操作

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2020-04-23-144842.jpg)



而成员变量信息则是存储在`class_ro_t`内部中的，我们来到`class_ro_t`内查看。
`class_rw_t` 表示**read write**，`class_ro_t` 表示 **read only**。

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2020-04-23-145011.jpg)



## 总结

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2020-04-23-145054.jpg)



一个类的内部结构如下,

![image-20190324153451659](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-24-085229.jpg)



![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-19-023218.jpg)



![image-20190324154051666](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-24-085240.jpg)

![image-20190324154130013](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-24-085336.jpg)



- superClass 指向当前类的父类
- cache_t 提供消息传递过程当中的缓存方法查找 ， 实质上是装满了 bucket_t 的一个 **hash 表**。因为散列表检索起来更快，
- class_data_bits_t 类的基础信息，包含了类的方法列表，协议列表等。



method_t是一个方法的封装,里面包括了名称(SEL),返回值,参数,与函数体(实现)