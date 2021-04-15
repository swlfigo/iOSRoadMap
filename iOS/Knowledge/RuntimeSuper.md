

# Father Son

### 下面的代码输出什么？

```objective-c
@implementation Son : Father
- (id)init {
    self = [super init];
    if (self) {
        NSLog(@"%@", NSStringFromClass([self class]));
        NSLog(@"%@", NSStringFromClass([super class]));
    }
    return self;
}
@end
```

 `self` 是类的隐藏参数,指向调用方法的这个类的实例，是一个 **指针**。
 而 `super` 跟 `self` 不一样,并不是指向父类的指针，只是一个 **编译器修饰符** 作用。

用 `self` 调用方法是从该类的方法列表当中找对应方法调用，如果没有就从父类当中找；而 `super` 关键词是从父类的方法列表当中找，调用父类的那个方法。但是这两种方式的事件调用者都是当前的实例 **Son** ,最终都是找到了 **NSObject** 中的 `class` 的方法。



在 NSObject.mm 中可以找到 -(Class)class 的实现：

```objective-c
- (Class)class {
    return object_getClass(self);
}
```

在 objc_class.mm 中找到 object_getClass 的实现：

```objective-c
Class object_getClass(id obj)
{
    if (obj) return obj->getIsa();
    else return Nil;
}
```

可以看到，最终这个方法返回的是，调用这个方法的 objc 的 isa 指针。那我们只需要知道在题干中的代码里面最终是谁在调用 -(Class)class 方法就可以找到答案了。

接下来，我们利用 **clang -rewrite-objc** 命令，将题干的代码转化为如下代码：

```objective-c
NSLog((NSString *)&__NSConstantStringImpl__var_folders_8k_cgm28r0d0bz94xnnrr606rf40000gn_T_Car_3f2069_mi_0, NSStringFromClass(((Class (*)(id, SEL))(void *)objc_msgSend)((id)self, sel_registerName("class"))));

NSLog((NSString *)&__NSConstantStringImpl__var_folders_8k_cgm28r0d0bz94xnnrr606rf40000gn_T_Car_3f2069_mi_1, NSStringFromClass(((Class (*)(__rw_objc_super *, SEL))(void *)objc_msgSendSuper)((__rw_objc_super){(id)self, (id)class_getSuperclass(objc_getClass("Car"))}, sel_registerName("class"))));
```

从上方可以得出，调用 Father class 的时候，本质是在调用

```js
objc_msgSendSuper(struct objc_super *super, SEL op, ...)
```

struct objc_super 的定义如下：

```js
struct objc_super {
    /// Specifies an instance of a class.
    __unsafe_unretained _Nonnull id receiver;

    /// Specifies the particular superclass of the instance to message. 
#if !defined(__cplusplus)  &&  !__OBJC2__
    /* For compatibility with old objc-runtime.h header */
    __unsafe_unretained _Nonnull Class class;
#else
    __unsafe_unretained _Nonnull Class super_class;
#endif
    /* super_class is the first class to search */
};
```



里面传两个参数，第一个参数objc_super结构体中有两个成员:

> - `receiver`: a pointer of type `id`. Specifies an instance of a class.
> - `super_class`: a pointer to a `Class` data structure. Specifies the particular superclass of the instance to the message.

`receiver` 就是调用这个事件的接受者 `self`，然后第二个就是父类的 `class`  **Father**，然后从这个 **Father** 类开始找 `class` 方法，一直找到了 **NSObject** ，最后这两个方法都是调用了 `[self class]` 打印当前类的 `class`。



从定义可以得知：当利用 super 调用方法时，只要编译器看到super这个标志，就会让当前对象去调用父类方法，本质还是当前对象在调用，是去父类找实现，super 仅仅是一个编译指示器。但是消息的接收者 receiver 依然是self。最终在 NSObject 获取 isa 指针的时候，获取到的依旧是 self 的 isa，所以，我们得到的结果是：Son。



## Reference

[Runtime学习：面试题狙击](https://cloud.tencent.com/developer/article/1449795)

[iOS:关于super 关键字(使用runtime分析)](https://www.jianshu.com/p/6da99ddc0b60)