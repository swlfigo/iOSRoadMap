# isa

## isa指针是什么?



isa指针保存着指向类对象的内存地址,类对象全局只有一个,因此每个类创建出来的对象都会默认有一个isa属性,保存类对象的地址,也就是class,通过class就可以查询到这个对象的属性和方法,协议等;



当 ObjC 为为一个对象分配内存，**初始化实例变量后**，在这些对象的实例变量的结构体中的第一个就是 `isa`。(isa 存储该对象信息,例如引用计数器，弱引用表等)

**注:** 有一些对象比较小则会使用 **TaggedPointer**技术,不使用isa



isa本质是一个isa_t的类型，那isa_t是一个**联合体位域结构**



```objective-c
union isa_t {
    isa_t() { }
    isa_t(uintptr_t value) : bits(value) { }

    Class cls;
    uintptr_t bits;
#if defined(ISA_BITFIELD)
    struct {
        ISA_BITFIELD;  // defined in isa.h
    };
#endif
};

```



**什么是联合体**？

**当多个数据需要共享内存或者多个数据每次只取其一时，可以利用联合体(union)，利用union可以用相同的存储空间存储不同型别的数据类型，从而节省内存空间。**



采用这种结构的原因也是基于内存优化的考虑（**即二进制中每一位均可表示不同的信息**）。通常来说，isa指针占用的内存大小是**8字节**，即**64位**，已经足够存储很多的信息了，这样可以极大的节省内存，以提高性能。



OC源码:

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2020-09-28-135637.jpg)

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2020-09-28-135651.jpg)

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2020-09-28-135839.jpg)

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2020-09-28-135907.jpg)



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



- **nonpointer**：表示是否对 isa 指针开启指针优化，0：纯isa指针，1：不⽌是类对象地址，isa 中包含了类信息、对象的引⽤计数等。 如果该实例对象启用了Non-pointer，那么会对isa的其他成员赋值，否则只会对cls赋值。

  

  是否关闭Non-pointer目前有这么几个判断条件，这些都可以在runtime源码objc-runtime-new.m中找到逻辑。

  ```objective-c
  1：包含swift代码；
  2：sdk版本低于10.11；
  3：runtime读取image时发现这个image包含__objc_rawisa段；
  4：开发者自己添加了OBJC_DISABLE_NONPOINTER_ISA=YES到环境变量中；
  5：某些不能使用Non-pointer的类，GCD等；
  6：父类关闭。
  ```

  

- **has_assoc**：关联对象标志位，0没有，1存在。

- **has_cxx_dtor**：该对象是否有 C++ 或者 Objc 的析构器，如果有析构函数，则需要做析构逻辑，如果没有，则可以更快的释放对象。

- **shiftcls**：存储类指针的值。开启指针优化的情况下，在 arm64 架构中有 33 位⽤来存储类指针。

- **magic**：⽤于调试器判断当前对象是真的对象还是没有初始化的空间。

- **weakly_referenced**：对象是否被指向或者曾经指向⼀个 ARC 的弱变量，没有弱引⽤的对象可以更快释放。

- **deallocating**：标志对象是否正在释放内存。

- **has_sidetable_rc**：当对象引⽤技术⼤于 10 时，则需要借⽤该变量存储进位。

- **extra_rc**：当表示该对象的引⽤计数值，实际上是引⽤计数值减 1，例如，如果对象的引⽤计数为 10，那么 extra_rc 为 9。如果引⽤计数⼤于 10，则需要使⽤到上⾯的 has_sidetable_rc。

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2020-11-26-145612.jpg)




整体如下图片所示：

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2020-09-28-135519.jpg)





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





## 为什么要设计metaclass

metaClass是单一职责和扩展性:  instance的信息由Class所有;  Class的信息则由metaClass所有;

否则类方法，实际方法都在同一个流程中，类对象、元类对象能够复用消息发送流程机制；

- 根据消息接受者的`isa`指针找到`metaclass`（因为类方法存在元类中。如果调用的是实例方法，isa指针指向的是类对象。） 
- 进入`CacheLookup`流程，这一步会去寻找方法缓存，如果缓存命中则直接调用方法的实现，如果缓存不存在则进入`objc_msgSend_uncached`流程。









## Reference

[从 NSObject 的初始化了解 isa](https://draveness.me/isa/)

[iOS底层探索：isa结构分析](https://juejin.im/post/6871047381450752013)