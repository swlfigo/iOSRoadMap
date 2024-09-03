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





## 什么是元类（meta-class）？

Objective-C 的一个类也是一个对象。这意味着你可以发送消息给一个类。

```
NSStringEncoding defaultStringEncoding = [NSString defaultStringEncoding];
```

在这个示例里，`defaultStringEncoding`被发送给了`NSString`类。

之所以能成功是因为 Objective-C 中每个类本身也是一个对象。如上面所看到的，这意味着类结构也必须以一个isa指针开始，从而可以和`objc_object`在二进制层面兼容，之后这个结构的下一字段必须是一个指向父类的指针（对于基类则为nil）。

正如我上周展示的，定义一个`Class`有很多种方式，取决于你的运行时库版本，但有一点，它们都以`isa`字段开始，并且仅跟着一个`superclass`字段。

```
typedef struct objc_class *Class;
struct objc_class {
    Class isa;
    Class super_class;
    /* followed by runtime specific details... */
};
```

为了调用`Class`里的方法，该`Class`的`isa`指针也必须指向一个包含了该`Class`方法列表的`Class`。

这就引出了元类的定义：元类是`Class`的类。

简单来说就是：

- 当你给对象发送消息时，消息是在寻找这个对象的类的方法列表;
- 当你给类发消息时，消息是在寻找这个类的元类的方法列表。

元类是必不可少的，因为它存储了类的类方法。每个类都必须有独一无二的元类，因为每个类都有独一无二的类方法。



## 元类的类是什么？

元类，就像之前的类一样，它也是一个对象。你也可以调用它的方法。自然的，这就意味着他必须也有一个类。

所有的元类都使用根元类（继承体系中处于顶端的类的元类）作为他们的类。这就意味着所有`NSObject`的子类（大多数类）的元类都会以`NSObject`的元类作为他们的类

根据这个规则，所有的元类使用根元类作为他们的类，根元类的元类则就是它自己。也就是说基类的元类的isa指针指向他自己。



## 类和元类的继承

类用`super_class`指针指向了父类，同样的，元类用`super_class`指向类的`super_class`的元类。

说的更拗口一点就是，根元类把它自己的基类设置成了`super_class`。

在这样的继承体系下，所有实例、类以及元类都继承自一个基类。

这意味着对于继承于`NSObject`的所有实例、类和元类，他们可以使用`NSObject`的所有实例方法，类和元类可以使用NSObject的所有类方法



## 为什么要设计metaclass

metaClass是单一职责和扩展性:  instance的信息由Class所有;  Class的信息则由metaClass所有;

否则类方法，实际方法都在同一个流程中，类对象、元类对象能够复用消息发送流程机制；

- 根据消息接受者的`isa`指针找到`metaclass`（因为类方法存在元类中。如果调用的是实例方法，isa指针指向的是类对象。） 
- 进入`CacheLookup`流程，这一步会去寻找方法缓存，如果缓存命中则直接调用方法的实现，如果缓存不存在则进入`objc_msgSend_uncached`流程。



## 类对象和元类对象分别是什么，他们之间有什么区别？



实例对象可以通过isa指针找到它的类对象，类对象存储实例方法列表等信息。类对象可以通过isa指针找到它的元类对象，从而可以访问类方法列表等相关信息



类对象或是元类对象都是objc_class数据结构的，objc_class由于继承自objc_object,所以他们都有isa指针，所有实例可以找到类，类可以找到元类




## Reference

[从 NSObject 的初始化了解 isa](https://draveness.me/isa/)

[iOS底层探索：isa结构分析](https://juejin.im/post/6871047381450752013)

[【译】Objective-C 中的元类（meta-class） | 土丘上的蒲公英 (dracarys.github.io)](https://dracarys.github.io/2018/07/31/What-is-a-mate-class-in-Objective-C/)