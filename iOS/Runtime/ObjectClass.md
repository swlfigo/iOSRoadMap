

# iOS 对象与数据结构

不管是类对象还是元类对象，类型都是 Class，class 和 mete-class 的底层都是 objc_class 结构体的指针。

```objective-c
typedef struct objc_class *Class;
```

不少网上文章都是复制粘贴错的,如下

\#if !**OBJC2** 在2006年7月WWDC中，Apple发布了“Objective-C 2.0”。2.0有很多的语法改进、runtime改进、垃圾回收机制（已废弃）、支持64 等。

上面“! **OBJC2** ” 之间的代码是Objective-C 2.0之前1.0版本的东西。2.0已经不支持了。

## 1.OBJC1 objc_class 结构（过时）

```cpp
struct objc_class {
    Class _Nonnull isa  OBJC_ISA_AVAILABILITY;

#if !__OBJC2__
    Class _Nullable super_class                              OBJC2_UNAVAILABLE;
    const char * _Nonnull name                               OBJC2_UNAVAILABLE;
    long version                                             OBJC2_UNAVAILABLE;
    long info                                                OBJC2_UNAVAILABLE;
    long instance_size                                       OBJC2_UNAVAILABLE;
    struct objc_ivar_list * _Nullable ivars                  OBJC2_UNAVAILABLE;
    struct objc_method_list * _Nullable * _Nullable methodLists                    OBJC2_UNAVAILABLE;
    struct objc_cache * _Nonnull cache                       OBJC2_UNAVAILABLE;
    struct objc_protocol_list * _Nullable protocols          OBJC2_UNAVAILABLE;
#endif

} OBJC2_UNAVAILABLE;
/* Use `Class` instead of `struct objc_class *` */
```

## 2. OBJC2 结构

**objc_class**的真实定义实际的代码我们可以从 [objc-runtime-new.h](https://opensource.apple.com/source/objc4/objc4-750.1/runtime/objc-runtime-new.h.auto.html) 中看到(中间代码省略)：

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2020-04-23-143952.jpg)



**objc_object**的真实定义 详见 [objc-private.h文件](https://opensource.apple.com/source/objc4/objc4-750.1/runtime/objc-private.h.auto.html)

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2020-04-23-144015.jpg)

如上图,关系也如旧版本一样, **objc_class** 继承于 **objc_object**

objc_object中有一个isa指针，那么objc_class继承objc_object，也就同样拥有一个isa指针



## 3. class_rw_t

**objc_class** 中的 data 返回 `class_rw_t` 结构，此结构定义如下：

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2020-04-23-144757.png)

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2020-04-23-144612.jpg)



而`class_rw_t`是通过bits调用data方法得来的，来到data方法内部实现。我们可以看到，data函数内部仅仅对bits进行`&FAST_DATA_MASK`操作

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2020-04-23-144842.jpg)



而成员变量信息则是存储在`class_ro_t`内部中的，我们来到`class_ro_t`内查看。
`class_rw_t` 表示**read write**，`class_ro_t` 表示 **read only**。

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2020-04-23-145011.jpg)



## 4. 总结

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