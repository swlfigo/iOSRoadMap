# weak的实现原理

> 使用场景都比较清晰，避免出现对象之间的强强引用而造成对象不能被正常释放最终导致内存泄露的问题。weak 关键字的作用是弱引用，所引用对象的计数器不会加1，并在引用对象被释放的时候自动被设置为 nil。



下面的一段代码是在开发中常见的weak的使用

```objective-c
Person *object = [[Person alloc]init];
id __weak objc = object;
```

此打断点跟踪汇编信息，可以发现底层库调了`objc_initWeak`函数

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2020-11-25-143524.jpg)



## objc_initWeak方法

下是`objc_initWeak`方法的底层源码

```
id objc_initWeak(id *location, id newObj)
{
    if (!newObj) {
        *location = nil;
        return nil;
    }

    return storeWeak<DontHaveOld, DoHaveNew, DoCrashIfDeallocating>
        (location, (objc_object*)newObj);
}
复制代码
```

该方法的两个参数`location`和`newObj`。

> - **location** ：**`__weak指针`的地址**，存储指针的地址，这样便可以在最后将其指向的对象置为nil。
> - **newObj** ：所引用的对象。即例子中的obj 。

`objc_initWeak`方法只是一个深层次函数调用的入口，在该方法内部调用了`storeWeak`方法。



## Reference

[1.iOS底层原理：weak的实现原理](https://juejin.cn/post/6844904101839372295)

