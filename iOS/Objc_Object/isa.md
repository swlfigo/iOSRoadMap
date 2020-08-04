# isa

**Objective-C 对象都是 C 语言结构体**，所有的对象都包含一个类型为 `isa` 的指针，不过现在的 ObjC 对象的结构已经不是这样了。代替 `isa` 指针的是结构体 `isa_t`, 这个结构体中"包含"了当前对象指向的类的信息

```objectivec
struct objc_object {
    isa_t isa;
};
```

当 ObjC 为为一个对象分配内存，**初始化实例变量后**，在这些对象的实例变量的结构体中的第一个就是 `isa`。

## Reference

[从 NSObject 的初始化了解 isa](https://draveness.me/isa/)

