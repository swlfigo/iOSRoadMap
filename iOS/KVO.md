### 5.KVO内部实现原理

* KVO是基于runtime机制实现的
* 当某个类的属性对象`第一次被观察`时，系统就会在运行期动态地创建该类的一个派生类，在这个派生类中重写基类中任何被观察属性的setter 方法。派生类在被重写的setter方法内实现真正的`通知机制`
* Apple使用 `isa混写(isa-swizzling)` 来实现KVO
* 如果原类为Person，那么生成的派生类名为NSKVONotifying_Person
* 每个类对象中都有一个isa指针指向当前类，当一个类对象的第一次被观察，那么系统会偷偷将isa指针指向动态生成的派生类，从而在给被监控属性赋值时执行的是派生类的setter方法
* 键值观察通知依赖于NSObject 的两个方法: `willChangeValueForKey:` 和 `didChangevlueForKey:` 在一个被观察属性发生改变之前， `willChangeValueForKey: `一定会被调用，这就会记录旧的值。而当改变发生后，`didChangeValueForKey:` 会被调用，继而 `observeValueForKey:ofObject:change:context: `也会被调用。
补充：KVO的这套实现机制中苹果还偷偷重写了class方法，让我们误认为还是使用的当前类，从而达到隐藏生成的派生类

![](http://img.isylar.com/media/15439069822328.jpg)



## Q:
1. 通过KVC设置value能否生效?

```objective-c
[xxx setValue:yyy forkey:zzz]
```
能生效

1. 通过成员变量直接复制value能否生效

```objective-c
//obj
-(void)increase{
    _value += 1;
}

//KVO
[obj increase];
```
这样手动调用方法改变，是不会触发KVO的,因为不会调用setter，如果需要KVO生效，需要手动KVO

```objective-c
-(void)increase{
//直接为成员变量赋值
    [self willChangeValueForKey:@"value"];
    _value += 1;
    [self didChangeValueForKey:@"value"];
}
```

## 总结
* 使用 `setter` 方法改变值 KVO 才会生效
* 使用 `setValue:forKey:` 改变 KVO 才会生效
* 成员变量直接修改需要 `手动添加` KVO 才会生效