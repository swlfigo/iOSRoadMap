# isa

**Objective-C 对象都是 C 语言结构体**，所有的对象都包含一个类型为 `isa` 的指针，不过现在的 ObjC 对象的结构已经不是这样了。代替 `isa` 指针的是结构体 `isa_t`, 这个结构体中"包含"了当前对象指向的类的信息

```objectivec
struct objc_object {
    isa_t isa;
};
```

当 ObjC 为为一个对象分配内存，**初始化实例变量后**，在这些对象的实例变量的结构体中的第一个就是 `isa`。

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

`isa_t` 是一个 `union` 类型的结构体,其中的 `isa_t`、`cls`、 `bits` 还有结构体共用同一块地址空间。而 `isa` 总共会占据 64 位的内存空间（决定于其中的结构体）

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2020-09-20-094253.jpg)

```objective-c
struct {
   uintptr_t indexed           : 1;
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



- 其中 `indexed` 表示 `isa_t` 的类型

  - 0 表示 `raw isa`，也就是没有结构体的部分，访问对象的 `isa` 会直接返回一个指向 `cls` 的指针
  - 1 表示当前 `isa` 不是指针，但是其中也有 `cls` 的信息，只是其中**关于类的指针都是保存在 `shiftcls` 中**

- has_cxx_dtor

  在设置 `indexed` 和 `magic` 值之后，会设置 `isa` 的 `has_cxx_dtor`，这一位表示当前对象有 C++ 或者 ObjC 的析构器(destructor)，如果没有析构器就会快速释放内存。

- has_assoc

  - 对象含有或者曾经含有关联引用，没有关联引用的可以更快地释放内存

- weakly_referenced

  - 对象被指向或者曾经指向一个 ARC 的弱变量，没有弱引用的对象可以更快释放

- deallocating

  - 对象正在释放内存

- has_sidetable_rc

  - 对象的引用计数太大了，存不下

- extra_rc

  - 对象的引用计数超过 1，会存在这个这个里面，如果引用计数为 10，`extra_rc` 的值就为 9



## Reference

[从 NSObject 的初始化了解 isa](https://draveness.me/isa/)

