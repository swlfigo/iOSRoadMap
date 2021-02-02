# Message Forwarding

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-19-061727.jpg)



当一个 OC 对象（receiver）接收到 Unknown selector 时，会进入如图流程，用户可以在这三个步骤中 override receiver 的相关方法，进而避免`doesNotRecognizeSelector:`异常。

《Effective Objective-C 2.0》的描述是：

> 步骤越往后，处理消息的代价就越大；最好能在第一步就处理完，这样的话，runtime 系统就可以将此方法缓存起来，进而提高效率。若想在第三步里把消息转发给备援的 receiver，那还不如把转发操作提前到第二步。因为第三步只是修改了调用目标，这项改动放在第二步会更为简单，不然的话，还得创建并处理完整的`NSInvocation`。

## +resolveInstanceMethod

Receiver 在收到 unknown selector 后，首先将调用其本类的`resolveInstanceMethod:`方法，该方法定义如下：

```objectivec
+ (BOOL)resolveInstanceMethod:(SEL)sel;
```

该方法的参数就是那个 unknown selector，其返回值为`Boolean`类型，表示这个类是否能新增一个实例方法用以处理该 unknown selector。在继续往下执行转发机制之前，本类有机会新增一个处理此 selector 的方法。所以`resolveInstanceMethod:`的一般使用套路是：

```objectivec
+ (BOOL)resolveInstanceMethod:(SEL)aSelector {
    if (/* aSelector满足某个条件  */) {
        /*
         调用class_addMethod为该类添加一个处理aSelector的方法，譬如：
         class_addMethod(self, aSelector, aImp, @"v@:@");
         */
        return YES;
    }
    return [super resolveInstanceMethod:aSelector];
}
```

假如尚未实现的方法不是实例方法而是类方法，那么 runtime 系统会调用另外一个与`resolveInstanceMethod:`类似的方法`resolveClassMethod:`。

就我经验而言，`resolveInstanceMethod:`的使用场景一般用来动态添加 setter 和 getter。

## [#](https://zhangbuhuai.com/post/message-forwarding.html#forwardingtargetforselector)-forwardingTargetForSelector

当前 receiver 还有第二次机会能处理 unknown selector，在这一步中，runtime 系统会问它：可否把这条消息转给其他对象处理？该步骤对应的处理方法是`forwardingTargetForSelector:`，定义于`<objc/NSObject.h>`中：

```objectivec
- (id)forwardingTargetForSelector:(SEL)aSelector;
```

若当前 receiver 能找到备援对象，则将其返回，当然，备援对象必须能够响应 aSelector，否则依然会抛出`doesNotRecognizeSelector:`异常；若找不到，则返回`nil`。

`-forwardingTargetForSelector:`的使用逻辑非常简单，应用场景包括：

- 实现多继承。Objective-C 不允许多继承，基于`-forwardingTargetForSelector:`，可以通过组合的方式，模拟出多继承的某些特性。
- 为协议遵循者提供默认实现。譬如某个协议定义了多个方法，有必要为这几个方法提供默认实现；具体做法是定义一个类（假设为 Implement），用于实现这几个方法，然后 override 协议遵循者的`-forwardingTargetForSelector:`方法，将协议方法的 receiver 定位到 Implement 对象。

## [#](https://zhangbuhuai.com/post/message-forwarding.html#forwardinvocation)-forwardInvocation

`-forwardInvocation:`要和`-methodSignatureForSelector:`配套使用，后者为`NSMethodSignature`对象，该对象携带 selector 的签名信息，包括参数类型、返回值类型和长度等。Runtime 内部会基于`NSMethodSignature`实例构建一个`NSInvocation`对象，作为回调`-forwardInvocation:`的入参。

只要回调`-methodSignatureForSelector:`的返回值不为空，就会进入`-forwardInvocation:`方法，用户可以在此过程中修改 invocation 的 target，将 receiver 定位到别处：

```objectivec
- (void)forwardInvocation:(NSInvocation *)invocation {
    [invocation setTarget:self.target];  // 让self.target成为消息的receiver
    [invocation invoke];
}
```

值得一提的是，除了修改 receiver，还可以修改入参，甚至是返回值。`NSInvocation#invoke`会触发 receiver 的 selector 的调用，如果不想调用怎么办？没怎么办，只要确保 invocation 的返回值（`NSInvocation#setReturnValue:`）的类型和长度一致即可。

Unknown selector 触发的三个回调介绍完毕，简单总结一下。

就作用而言，`+resolveInstanceMethod:`主要用于为类动态增加实例方法；`-forwardingTargetForSelector:`用于将 selector 的 receiver 从`self`定位到别的 target；这两个方法的使用都比较直接简单，不太能整出花样。`-forwardInvocation:`就不同了，在它身上可以动的手脚比较多，不光可以修改 receiver，还可以篡改入参、返回值；当然，`-forwardInvocation:`的代价比较大一些，毕竟还会触发`-methodSignatureForSelector:`，构建`NSMethodSignature`和`NSInvocation`实例。

如果需要动态新增方法，可以在`+resolveInstanceMethod:`阶段完成；如果只是需要篡改 receiver，在`-forwardingTargetForSelector:`阶段完成更省事儿；如果需要更高阶的玩法，或许真的只有`-forwardInvocation:`能满足需求。



## Reference

[1.NSObject 的消息转发机制](https://zhangbuhuai.com/post/message-forwarding.html)