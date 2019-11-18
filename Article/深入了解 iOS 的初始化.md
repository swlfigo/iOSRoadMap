> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 https://juejin.im/post/5dd24e3ff265da0bbc3067ae



# 深入了解 iOS 的初始化.md

初始化
---

在 iOS 里面，无论是 Objective-C 还是 Swift，类（结构体、枚举）的初始化都有一定的规则要求，只不过在 Objective-C 中会比较宽松，如果不按照规则也不会报错，但会存在隐患，而在 Swift 则需要严格按照规则要求代码才能编译通过，极大提高了代码的安全性。

类（结构体、枚举）的初始化有两种初始化器（初始化方法）：指定初始化器（Designated Initializers ）、便利初始化器（Convenience Initializers）

### Designated Initializers

指定初始化器是类（结构体、枚举）的主初始化器，类（结构体、枚举）初始化的时候必须调用自身或者父类的指定初始化器。一个类（结构体、枚举）可以有多个指定初始化器，作用是代表从不同的源进行初始化。一个类（结构体、枚举）除非有多种不同的源进行初始化，否则不建议创建多个指定初始化器。在 iOS 里，视图控件类，如果：`UIView`、`UIViewController`就有两个指定初始化器，分别代表从代码初始化、从`Nib`初始化

### Convenience Initializers

便利初始化器是类（结构体、枚举）的次要初始化器，作用是使类（结构体、枚举）在初始化时更方便设置相关的属性（成员变量）。既然便利初始化器是为了便利，那么一个类（结构体、枚举）就可以有多个便利初始化器，这些便利初始化器里面最后都需要调用自身的指定初始化器

### 核心规则

iOS 的初始化最核心两条的规则：

*   必须至少有一个指定初始化器，在指定初始化器里保证所有非可选类型属性都得到正确的初始化（有值）
*   便利指定初始化器必须调用其他初始化器，使得最后肯定会调用指定初始化器

![](https://user-gold-cdn.xitu.io/2019/11/18/16e7d819f4efa87d?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

所有的其他规则都根据这两条规则而展开，只是 Objective-C 没有那么多安全检查，显得比较随意、宽松，而 Swift 则有一堆的限制。

Objective-C
-----------

Objective-C 在初始化时，会自动给每个属性（成员变量）赋值为 0 或者 `nil`，没有强制要求额外为每个属性（成员变量）赋值，方便的同时也缺少了代码的安全性。

Objective-C 中的指定初始化器会在后面被`NS_DESIGNATED_INITIALIZER`修饰，以下为`NSObject` 和`UIView`的指定初始化器

```
// NSObject
@interface NSObject <NSObject> 

- (instancetype)init
#if NS_ENFORCE_NSOBJECT_DESIGNATED_INITIALIZER
    NS_DESIGNATED_INITIALIZER
#endif
    ;
@end
  
  
// UIView
@interface UIView : UIResponder

- (instancetype)initWithFrame:(CGRect)frame NS_DESIGNATED_INITIALIZER;
- (nullable instancetype)initWithCoder:(NSCoder *)coder NS_DESIGNATED_INITIALIZER;

@end
复制代码

```

在 Objective-C 里面，所有类都继承自`NSObject`。当自定义一个类的时候，要么直接继承自`NSObject`，要么继承自`UIView`或者其他类。

无论继承自什么类，都经常需要新的初始化方法，而这个新的初始化方法其实就是新的指定初始化器。如果存在一个新的指定初始化器，那么原来的指定初始化器就会自动退化成便利初始化器。为了遵循必须要调用指定初始化器的规则，就必须重写旧的定初始化器，在里面调用新的指定初始化器，这样就能确保所有属性（成员变量）被初始化

根据这条规则，可以从`NSObject`、`UIView`中看出，由于`UIView`拥有新的指定初始化器`-initWithFrame:`，导致父类`NSObject`的指定初始化器`-init`退化成便利初始化器。所以当调用`[[UIView alloc] init]`时，`-init`里面必然调用了`-initWithFrame:`

当存在一个新的指定初始化器的时候，推荐在方法名后面加上`NS_DESIGNATED_INITIALIZER`，主动告诉编译器有一个新的指定初始化器，这样就可以使用 XCode 自带的`Analysis`功能分析，找出初始化过程中可能存在的漏洞

```
@interface MyView : UIView

@property (nonatomic, strong) NSString *name;

// 推荐加上NS_DESIGNATED_INITIALIZER
- (instancetype)initWithFrame:(CGRect)frame name:(NSString *)name NS_DESIGNATED_INITIALIZER;

@end


@implementation MyView

// 初始化时加入参数name，这个方法已经成为新的指定初始化器
- (instancetype)initWithFrame:(CGRect)frame name:(NSString *)name {
    if (self = [super initWithFrame:frame]) {
        self.name = name;
    }
    return self;
}

// 旧的指定初始化器就自动退化成便利初始化器，必须在里面调用新的指定初始化器
- (instancetype)initWithFrame:(CGRect)frame {
    return [self initWithFrame:frame name:@"Daniels"];
}

@end
复制代码

```

当然，一个新的类也可以不增加新的初始化方法，在 Objective-C 中，子类会直接继承父类所有的初始化方法

Swift
-----

在 Swift 中，初始化器的规则严格且复杂，目的就是为了使代码更加安全，如果不符合规则，会直接报错，常常会让刚接手 Swift 或者一直对 iOS 的初始化没有深入理解的人很头疼。其实核心规则还是一样，只要理解了各个规则的含义和作用，写起来还是没有压力。

从 iOS 初始化的核心规则展开而来，Swift 多了一些规则：

*   初始化的时候需要保证类（结构体、枚举）的所有非可选类型属性都会有值，否则会报错。
*   在没有给所有非可选类型属性赋值（初始化完成）之前，不能调用`self`相关的任何东西，例如：调用实例属性，调用实例方法。

### 不存在继承

这种情况处理就十分简单，自己里面的`init`方法就是它的指定初始化器，而且可以随意创建多个它的指定初始化器。如果需要创建便利初始化器，则在方法名前面加上`convenience`，且在里面必须调用其他初始化器，使得最后肯定调用指定初始化器

```
class Person {

    var name: String

    var age: Int

    // 可以存在多个指定初始化器
    init(name: String, age: Int) {
        self.name = name;
        self.age = age;
    }

    // 可以存在多个指定初始化器
    init(age: Int) {
        self.name = "Daniels";
        self.age = age;
    }

    // 便利初始化器
    convenience init(name: String) {
        // 必须要调用自己的指定初始化器
        self.init(name: name, age: 18)
        // 必须在初始化完成后才能调用实例方法
        jump()
    }
  
    func jump() {

    }
}
复制代码

```

### 存在继承

如果子类没有新的非可选类型属性，或者保证所有非可选类型属性都已经有默认值，则可以直接继承父类的指定初始化器和便利初始化器

```
class Student: Person {

    var score: Double = 100
  
}
复制代码

```

如果子类有新的非可选类型属性，或者无法保证所有非可选类型属性都已经有默认值，则需要新创建一个指定初始化器，或者重写父类的指定初始化器

*   新创建一个指定初始化器，会覆盖父类的指定初始化器，需要先给当前类所有非可选类型属性赋值，然后再调用父类的指定初始化器
*   重写父类的指定初始化器，需要先给当前类所有非可选类型属性赋值，然后再调用父类的指定初始化器
*   在保证子类有指定初始化器，才能创建便利初始化器，且在便利初始化器里面必须调用指定初始化器

```
class Student: Person {

    var score: Double
		
    // 新的指定初始化器，如果有新的指定初始化器，就不会继承父类的所有初始化器，除非重写
    init(name: String, age: Int, score: Double) {
        self.score = score
        super.init(name: name, age: age)
    }
  
    // 重写父类的指定初始化器，如果不重写，则子类不存在这个方法
    override init(name: String, age: Int) {
        score = 100
        super.init(name: name, age: age)
    }
  
  
    // 便利初始化器
    convenience init(name: String) {
        // 必须要调用自己的指定初始化器
        self.init(name: name, age: 10, score: 100)
    }
}
复制代码

```

需要注意的是，如果子类重写父类所有指定初始化器，则会继承父类的便利初始化器。原因也是很简单，因为父类的便利初始化器，依赖于自己的指定初始化器

### Failable Initializers

在 Swift 中可以定义一个可失败的初始化器（Failable Initializers），表示在某些情况下会创建实例失败。

只有在表示创建失败的时候才有返回值，并且返回值为`nil`。

子类可以把父类的可失败的初始化器重写为不可失败的初始化器，但不能把父类的不可失败的初始化器重写为可失败的初始化器

```
class Animal {
    
    let name: String
    // 可失败的初始化器，如果把 ! 换成 ?，则为隐式的可失败的初始化器
    init?(name: String) {
        if name.isEmpty {
            return nil
        }
        self.name = name
    }
}

class Dog: Animal {

    override init(name: String) {
        if name.isEmpty {
            super.init(name: "旺财")!
        } else {
            super.init(name: name)!
        }
    }
}
复制代码

```

### Required Initializers

在 Swift 中，可以使用`required`修饰初始化器，来指定子类必须实现该初始化器。需要注意的是，如果子类可以直接继承父类的指定初始化器和便利初始化器，所以也就可以不用额外实现`required`修饰的初始化器

子类实现该初始化器时，也必须加上`required`修饰符，而不是`override`

```
class MyView: UIView {

    var name: String


    init(frame: CGRect, name: String) {
        self.name = name;
        super.init(frame: frame)
    }

    // 必须实现此初始化器，但由于是可失败的初始化器，所以里面可以不做具体实现
    required init?(coder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }
}
复制代码

```

总结
--

iOS 的初始化最核心两条的规则：

*   必须至少有一个指定初始化器，在指定初始化器里保证所有非可选类型属性都得到正确的初始化（有值）
*   便利指定初始化器必须调用其他初始化器，使得最后肯定会调用指定初始化器

展开而来的多条规则：

*   无论在 Objective-C 还是 Swift 中，都可以有多个指定初始化器和多个便利指定初始化器。如果不是可以从多个不同的源初始化，最好只创建一个指定初始化器
*   无论在 Objective-C 还是 Swift 中，都需要在便利初始化器中调用指定初始化器
*   在 Objective-C 中，初始化的时候不需要保证所有属性（成员变量）都有值
*   在 Objective-C 中，如果存在一个新的指定初始化器，那么原来的指定初始化器就会自动退化成便利初始化器。必须重写旧的定初始化器，在里面调用新的指定初始化器
*   在 Swift 中，初始化的时候需要保证类（结构体、枚举）的所有非可选类型属性都会有值
*   在 Swift 中，必须在初始化完成后才能调用实例属性，调用实例方法
*   在 Swift 中，如果存在继承，并且子类有新的非可选类型属性，或者无法保证所有非可选类型属性都已经有默认值，那么就需要新创建一个指定初始化器，或者重写父类的指定初始化器，并且在里面调用父类的指定初始化器
*   在 Swift 中，子类如果没有新创建一个指定初始化器，并且没有重写父类的指定初始化器，则会继承父类的指定初始化器和便利指定初始化器
*   在 Swift 中，子类如果新创建一个指定初始化器，或者重写了父类的某个指定初始化器，那么就不会继承父类的指定初始化器和便利指定初始化器；但是如果重写了父类的所有指定初始化器，就会继承父类的便利初始化器
*   在 Swift 中，子类可以把父类的指定初始化器重写成便利初始化器
*   在 Swift 中，如果子类没有直接继承父类的指定初始化器和便利指定初始化器，则必须实现父类中`required`修饰的初始化器

参考资料
----

[Initialization](https://docs.swift.org/swift-book/LanguageGuide/Initialization.html)