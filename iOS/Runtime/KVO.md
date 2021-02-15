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

#### 


作者：我是好宝宝
链接：https://juejin.cn/post/6844904090569277447
来源：掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。