# KVO



## KVO初探

`KVO（Key-Value Observing）`是苹果提供的一套事件通知机制，这种机制允许将其他对象的特定属性的更改通知给对象。iOS开发者可以使用`KVO` 来检测对象属性的变化、快速做出响应，这能够为我们在开发强交互、响应式应用以及实现视图和模型的双向绑定时提供大量的帮助。

在[Documentation Archieve](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/KeyValueObserving/KeyValueObserving.html)中提到一句想要理解`KVO`，必须先理解`KVC`，因为`键值观察`是建立在`键值编码`的基础上

> In order to understand key-value observing, you must first understand key-value coding.——Key-Value Observing Programming Guide

而`KVO`和`NSNotificatioCenter`都是iOS观察者模式的一种实现，两者的区别在于：

- 相对于被观察者和观察者之间的关系，`KVO`是一对一的，`NSNotificatioCenter`是一对多的
- `KVO`对被监听对象无侵入性，不需要修改其内部代码即可实现监听

## KVO使用及注意点

#### 1.基本使用

KVO使用三部曲：

- 注册观察者

```objective-c
[self.person addObserver:self forKeyPath:@"name" options:(NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld) context:NULL];
```

- 实现回调

```objective-c
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSKeyValueChangeKey,id> *)change context:(void *)context {
    if ([keyPath isEqualToString:@"name"]) NSLog(@"%@", change);
}
```

- 移除观察者

```objective-c
[self.person removeObserver:self forKeyPath:@"name"];
```

#### 2.context的使用

`Key-Value Observing Programming Guide`是这么描述`context`的

> ![img](https://user-gold-cdn.xitu.io/2020/3/14/170d77a1fa7cf438?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)
>
> 消息中的上下文指针包含任意数据，这些数据将在相应的更改通知中传递回观察者；您可以指定NULL并完全依赖键路径字符串来确定更改通知的来源，但是这种方法可能会导致对象的父类由于不同的原因而观察到相同的键路径，因此可能会出现问题；一种更安全，更可扩展的方法是使用上下文确保您收到的通知是发给观察者的，而不是超类的。

这里提出一个假想，如果父类中有个`name`属性，子类中也有个`name`属性，两者都注册对`name`的观察，那么仅通过`keyPath`已经区分不了是哪个`name`发生变化了，现有两个解决办法：

- 多加一层判断——判断`object`，显然为了满足业务需求而去增加逻辑判断是不可取的
- 使用`context`传递信息，更安全、更可扩展

**`context`使用总结:**

- 不使用context作为观察值

```objective-c
// context是 void * 类型，应该填 NULL 而不是 nil
[self.person addObserver:self forKeyPath:@"name" options:(NSKeyValueObservingOptionNew) context:NULL];
```

- 使用context传递信息

```objective-c
static void *PersonNameContext = &PersonNameContext;
static void *ChildNameContext = &ChildNameContext;

[self.person addObserver:self forKeyPath:@"name" options:(NSKeyValueObservingOptionNew) context:PersonNameContext];
[self.child addObserver:self forKeyPath:@"name" options:(NSKeyValueObservingOptionNew) context:ChildNameContext];

- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSKeyValueChangeKey,id> *)change context:(void *)context {
    if (context == PersonNameContext) {
        NSLog(@"%@", change);
    } else if (context == ChildNameContext) {
        NSLog(@"%@", change);
    }
}
```



## KVO原理——isa-swizzling

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-02-16-135638.jpg)

`Key-Value Observing Programming Guide`中有一段底层实现原理的叙述



- KVO是使用`isa-swizzling`技术实现的
- 顾名思义，isa指针指向维护分配表的对象的类，该分派表实质上包含指向该类实现的方法的指针以及其他数据
- 在为对象的属性注册观察者时，将修改观察对象的isa指针，指向中间类而不是真实类。isa指针的值不一定反映实例的实际类
- 您永远不应依靠isa指针来确定类成员身份。相反，您应该使用class方法来确定对象实例的类

- 注册观察者之前：类对象为

  ```
  FXPerson
  ```

  ，实例对象isa指向

  ```
  FXPerson
  ```

  ![img](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-02-16-135759.jpg)

- 注册观察者之后：类对象为 `FXPerson`，实例对象isa指向 `NSKVONotifying_FXPerson`

  ![img](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-02-16-135804.jpg)

从这两图中可以得出一个结论：观察者注册前后`FXPerson类`没发生变化，但实例对象的`isa`指向发生变化





对观察者模式的又一实现； Apple 使用了 **isa 混写**(isa-swizzling) 来实现KVO.



- KVO是基于runtime机制实现的
- 当某个类的属性对象`第一次被观察`时，系统就会在运行期动态地创建该类的一个派生类，在这个派生类中重写基类中任何被观察属性的setter 方法。派生类在被重写的setter方法内实现真正的`通知机制`

- 每个类对象中都有一个isa指针指向当前类，当一个类对象的第一次被观察，那么系统会偷偷将isa指针指向动态生成的派生类，从而在给被监控属性赋值时执行的是派生类的setter方法
- KVO的这套实现机制中苹果还偷偷重写了class方法，让我们误认为还是使用的当前类，从而达到隐藏生成的派生类

![](https://sylarimage.oss-cn-shenzhen.aliyuncs.com/20190318144841.png)



当调用了 `addObserver:forkeypath`方法之后，系统会动态创建 `NSKVONorifying_A`类，同时将A的isa指针指向 NSKVONorifying_A。

iOS中`KVO`分为自动监听和手动触发两种形式

手动触发:

1. 重写监听属性的set、get方法
2. 重写 `+ (BOOL)automaticallyNotifiesObserversForKey:(NSString *)key` 
3. 在set方法中在赋值的前后分别调用：`willChangeValueForKey`和`didChangeValueForKey`
4. 实现`willChangeValueForKey`和`didChangeValueForKey`方法

其中需要重写

```objective-c
+ (BOOL)automaticallyNotifiesObserversForKey:(NSString *)key{
  if([key isEqualToString:@"age"]){
  //取消自动发送通知
    return NO;
  }else{
    return [super automaticallyNotifiesObserversForKey:key];
  }
}
```

## 总结

1. `automaticallyNotifiesObserversForKey`为`YES`时注册观察属性会生成动态子类`NSKVONotifying_XXX`
2. 动态子类观察的是`setter`方法
3. 动态子类重写了观察属性的`setter`方法 `dealloc` `class` `_isKVOA`方法
   - `setter`方法用于观察键值
   - `dealloc`方法用于释放时对isa指向进行操作
   - `class`方法用于指回动态子类的父类
   - `_isKVOA`用来标识是否是在观察者状态的一个标志位
4. `dealloc`之后`isa`指向元类
5. `dealloc`之后动态子类不会销毁



## Reference

[1 iOS探索 KVO原理及自定义](https://juejin.cn/post/6844904090569277447)