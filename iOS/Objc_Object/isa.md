# isa

**Objective-C 对象都是 C 语言结构体**，所有的对象都包含一个类型为 `isa` 的指针，不过现在的 ObjC 对象的结构已经不是这样了。代替 `isa` 指针的是结构体 `isa_t`, 这个结构体中"包含"了当前对象指向的类的信息

```objectivec
struct objc_object {
    isa_t isa;
};
```

当 ObjC 为为一个对象分配内存，**初始化实例变量后**，在这些对象的实例变量的结构体中的第一个就是 `isa`。

isa本质是一个isa_t的类型，那么isa_t类型又是怎么样的呢？继续探索，我们可以发现，其实isa_t是一个**联合体位域结构**，采用这种结构的原因也是基于内存优化的考虑（**即二进制中每一位均可表示不同的信息**）。通常来说，isa指针占用的内存大小是**8字节**，即**64位**，已经足够存储很多的信息了，这样可以极大的节省内存，以提高性能。



OC源码:

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2020-09-28-135637.jpg)

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2020-09-28-135651.jpg)

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2020-09-28-135839.jpg)

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2020-09-28-135907.jpg)



## isa 指针的作用与元类

**Objective-C 中类也是一个对象**。

因为在 Objective-C 中，对象的方法并**没有存储于对象的结构体中**（如果每一个对象都保存了自己能执行的方法，那么对内存的占用有极大的影响）。

当**实例方法**被调用时，它要通过自己持有的 `isa` 来查找对应的类，然后在这里的 `class_data_bits_t` 结构体中查找对应方法的实现。同时，每一个 `objc_class` 也有一个**指向自己的父类的指针** `super_class` 用来查找继承的方法。



类方法的实现又是如何查找并且调用的呢？这时，就需要引入*元类*来保证无论是类还是对象都能**通过相同的机制查找方法的实现**。

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2020-09-20-074153.jpg)

让每一个类的 `isa` 指向对应的元类，这样就达到了使类方法和实例方法的调用机制相同的目的：

- 实例方法调用时，通过对象的 `isa` 在类中获取方法的实现
- 类方法调用时，通过类的 `isa` 在元类中获取方法的实现



![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2020-09-20-074337.jpg)



## 结构体 isa_t

`isa_t` 是一个 `union` 类型的结构体,其中的 `isa_t`、`cls`、 `bits` 还有结构体共用同一块地址空间。而 `isa` 总共会占据 64 位的内存空间, 8 字节（决定于其中的结构体）

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2020-09-20-094253.jpg)

```objective-c
struct {
   uintptr_t nonpointer        : 1;
   uintptr_t has_assoc         : 1;
   uintptr_t has_cxx_dtor      : 1;
   uintptr_t shiftcls          : 44;
   uintptr_t magic             : 6;
   uintptr_t weakly_referenced : 1;
   uintptr_t deallocating      : 1;
   uintptr_t has_sidetable_rc  : 1;
   uintptr_t extra_rc          : 8;
};
```



- **nonpointer**：表示是否对 isa 指针开启指针优化，0：纯isa指针，1：不⽌是类对象地址，isa 中包含了类信息、对象的引⽤计数等。
- **has_assoc**：关联对象标志位，0没有，1存在。
- **has_cxx_dtor**：该对象是否有 C++ 或者 Objc 的析构器，如果有析构函数，则需要做析构逻辑，如果没有，则可以更快的释放对象。
- **shiftcls**：存储类指针的值。开启指针优化的情况下，在 arm64 架构中有 33 位⽤来存储类指针。
- **magic**：⽤于调试器判断当前对象是真的对象还是没有初始化的空间。
- **weakly_referenced**：志对象是否被指向或者曾经指向⼀个 ARC 的弱变量，没有弱引⽤的对象可以更快释放。
- **deallocating**：标志对象是否正在释放内存。
- **has_sidetable_rc**：当对象引⽤技术⼤于 10 时，则需要借⽤该变量存储进位。
- **extra_rc**：当表示该对象的引⽤计数值，实际上是引⽤计数值减 1，例如，如果对象的引⽤计数为 10，那么 extra_rc 为 9。如果引⽤计数⼤于 10，则需要使⽤到上⾯的 has_sidetable_rc。


整体如下图片所示：

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2020-09-28-135519.jpg)



## Reference

[从 NSObject 的初始化了解 isa](https://draveness.me/isa/)

[iOS底层探索：isa结构分析](https://juejin.im/post/6871047381450752013)