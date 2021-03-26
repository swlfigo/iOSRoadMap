# FluentDarkModeKit 

## NSProxy

NSProxy 是少数不继承自 NSObject 的类型。

在该框架中 NSProxy 承载了两种模式下的不同颜色和不同图片。

## 颜色 UIColor

`FluentDarkModeKit` 声明了`DMDynamicColor` 类，

```objective-c
NS_SWIFT_NAME(DynamicColor)
@interface DMDynamicColor : UIColor

@property (nonatomic, readonly) UIColor *lightColor;
@property (nonatomic, readonly) UIColor *darkColor;

- (instancetype)initWithLightColor:(UIColor *)lightColor darkColor:(UIColor *)darkColor;

@end
```

在 .h 文件中，我们可以看出 DMDynamicColor 继承子 UIColor，但是在 .m 中，我们可以看出它真正创建的是一个 `DMDynamicColorProxy`。

```objective-c
@interface DMDynamicColorProxy : NSProxy <NSCopying>

- (UIColor *)initWithLightColor:(UIColor *)lightColor darkColor:(UIColor *)darkColor {
  return (DMDynamicColor *)[[DMDynamicColorProxy alloc] initWithLightColor:lightColor darkColor:darkColor];
}

@end
```

DMDynamicColorProxy 继承自 `NSProxy`，它将所有的事件转发到 `resolvedColor` ，而 `resolvedColor` 是根据当前系统的模式返回的 `lightColor` 或者 `darkColor`。这样 DMDynamicColorProxy 对外的表现就是一个 UIColor，并且可以根据系统的模式返回对应的颜色。

```objective-c
@interface DMDynamicColorProxy : NSProxy <NSCopying>

@property (nonatomic, strong) UIColor *lightColor;
@property (nonatomic, strong) UIColor *darkColor;

@property (nonatomic, readonly) UIColor *resolvedColor;

@end

@implementation DMDynamicColorProxy

// TODO: We need a more generic initializer.
- (instancetype)initWithLightColor:(UIColor *)lightColor darkColor:(UIColor *)darkColor {
  self.lightColor = lightColor;
  self.darkColor = darkColor;

  return self;
}

- (UIColor *)resolvedColor {
  if (DMTraitCollection.currentTraitCollection.userInterfaceStyle == DMUserInterfaceStyleDark) {
    return self.darkColor;
  } else {
    return self.lightColor;
  }
}

// MARK: UIColor

- (UIColor *)colorWithAlphaComponent:(CGFloat)alpha {
  return [[DMDynamicColor alloc] initWithLightColor:[self.lightColor colorWithAlphaComponent:alpha]
                                          darkColor:[self.darkColor colorWithAlphaComponent:alpha]];
}

// MARK: NSProxy

- (NSMethodSignature *)methodSignatureForSelector:(SEL)sel {
  return [self.resolvedColor methodSignatureForSelector:sel];
}

- (void)forwardInvocation:(NSInvocation *)invocation {
  [invocation invokeWithTarget:self.resolvedColor];
}

// MARK: NSObject

- (BOOL)isKindOfClass:(Class)aClass {
  static DMDynamicColor *dynamicColor = nil;
  static dispatch_once_t onceToken;
  dispatch_once(&onceToken, ^{
    dynamicColor = [[DMDynamicColor alloc] init];
  });
  return [dynamicColor isKindOfClass:aClass];
}

// MARK: NSCopying

- (id)copy {
  return [self copyWithZone:nil];
}

- (id)copyWithZone:(NSZone *)zone {
  return [[DMDynamicColorProxy alloc] initWithLightColor:self.lightColor darkColor:self.darkColor];
}

@end
```

**注意：对于 UIColor 的方法中返回值为 UIColor 的，DMDynamicColorProxy 都进行了实现，目的就是当 UIColor 在调用这些方法时，返回的类型依然为 DMDynamicColorProxy。**

## 图片 UIImage

和颜色的实现原理一样，也声明了 DMDynamicImageProxy，由 resolvedImage 根据当前的模式返回 lightImage 或者 darkImage。

```objective-c
@interface DMDynamicImageProxy : NSProxy

@property (nonatomic, readonly) UIImage *resolvedImage;

- (instancetype)initWithLightImage:(UIImage *)lightImage darkImage:(UIImage *)darkImage;

@end
```

在具体的实现中，DMDynamicImageProxy 也是将事件转发到 resolvedImage，这样在外界看来 DMDynamicImageProxy 的表现就是 UIImage，但是可以根据当前的模式返回不同的 Image。

**注意：对于 UIImage 的方法中返回值为 UIImage 的，DMDynamicImageProxy 都进行了实现，目的就是当 UIImage 在调用这些方法时，返回的类型依然为 DMDynamicImageProxy。**

## 替换设置方法

我们先来看一个小测试，同一个颜色（实际类型为 DMDynamicColorProxy）赋值给 view 的 backgroundColor 和 button 的 titleColor 后，再和原来的颜色进行对比，结果是否相等？

```objective-c
let color = UIColor(.dm, light: .white, dark: .black)
view.backgroundColor = color
if view.backgroundColor == color {
  debugPrint("equal")
} else {
  debugPrint("not equal")
}

let button = UIButton()
button.setTitleColor(color, for: .normal)
if button.titleColor(for: .normal) == color {
  debugPrint("equal")
} else {
  debugPrint("not equal")
}
```

输出：

```shell
not equal
equal
```

也就是说，同样是给颜色进行赋值，但是 Apple 的处理是不一样的，有的和被赋予的值一致，有的则不一致。（应该是有些赋值会对颜色进行拷贝）

如果使用 DMDynamicColorProxy 对一个颜色进行赋值，再取出时类型却变成 UIColor 的，它就丢失了 lightColor 和 darkColor。对于这种属性设置，需要在设置 DMDynamicColorProxy 时进行保存。

所以 `FluentDarkModeKit` 对这类的属性进行了替换，例如 `setTintColor`：

```swift
extension UIView {
  private struct Constants {
    static var dynamicTintColorKey = "dynamicTintColorKey"
  }

  // 转化 setter: tintColor 的方法
  // 设置的时候，记录 dm_dynamicTintColor
  static let swizzleSetTintColorOnce: Void = {
    if !dm_swizzleInstanceMethod(#selector(setter: tintColor), to: #selector(dm_setTintColor)) {
      assertionFailure(DarkModeManager.messageForSwizzlingFailed(class: UIView.self, selector: #selector(setter: tintColor)))
    }
  }()

  private var dm_dynamicTintColor: DynamicColor? {
    get {
      return objc_getAssociatedObject(self, &Constants.dynamicTintColorKey) as? DynamicColor
    }
    set {
      objc_setAssociatedObject(self, &Constants.dynamicTintColorKey, newValue, .OBJC_ASSOCIATION_COPY_NONATOMIC)
    }
  }

  @objc private dynamic func dm_setTintColor(_ color: UIColor) {
    dm_dynamicTintColor = color as? DynamicColor
    dm_setTintColor(color)
  }
}
```

## 其他方法的替换

### willMove(toWindow:)

页面上显示的 view 可以通过 subviews，一层一层的获取到，然后根据当前的模式进行修改颜色。对于不在页面上显示的 view，只能通过替换 `willMove(toWindow:)` 方法，在添加到 window 时更新当前模式对应的颜色和图片。

```swift
extension UIView {
  // 调用 willMove(toWindow:) 的时候：
  // 1. dm_updateDynamicColors
  // 2. dm_updateDynamicImages
  static let swizzleWillMoveToWindowOnce: Void = {
    if !dm_swizzleInstanceMethod(#selector(willMove(toWindow:)), to: #selector(dm_willMove(toWindow:))) {
      assertionFailure(DarkModeManager.messageForSwizzlingFailed(class: UIView.self, selector: #selector(willMove(toWindow:))))
    }
  }()

  @objc private dynamic func dm_willMove(toWindow window: UIWindow?) {
    dm_willMove(toWindow: window)
    if window != nil {
      dm_updateDynamicColors()
      dm_updateDynamicImages()
    }
  }
}
```

### setBackgroundColor

替换 setBackgroundColor 有点特殊，替换代码如下：

```objective-c
@implementation UIView (DarkModeKit)

static void (*dm_original_setBackgroundColor)(UIView *, SEL, UIColor *);


/// 设置背景色
static void dm_setBackgroundColor(UIView *self, SEL _cmd, UIColor *color) {
  // 记录
  if ([color isKindOfClass:[DMDynamicColor class]]) {
    self.dm_dynamicBackgroundColor = (DMDynamicColor *)color;
  } else {
    self.dm_dynamicBackgroundColor = nil;
  }
  // 设置
  dm_original_setBackgroundColor(self, _cmd, color);
}

// https://stackoverflow.com/questions/42677534/swizzling-on-properties-that-conform-to-ui-appearance-selector
+ (void)dm_swizzleSetBackgroundColor {
  static dispatch_once_t onceToken;
  dispatch_once(&onceToken, ^{
    Method method = class_getInstanceMethod(self, @selector(setBackgroundColor:));
    dm_original_setBackgroundColor = (void *)method_getImplementation(method);
    method_setImplementation(method, (IMP)dm_setBackgroundColor);
  });
}

- (DMDynamicColor *)dm_dynamicBackgroundColor {
  return objc_getAssociatedObject(self, _cmd);
}

- (void)setDm_dynamicBackgroundColor:(DMDynamicColor *)dm_dynamicBackgroundColor {
  objc_setAssociatedObject(self,
                           @selector(dm_dynamicBackgroundColor),
                           dm_dynamicBackgroundColor,
                           OBJC_ASSOCIATION_COPY_NONATOMIC);
}

@end

```

# 命名空间

`FluentDarkModeKit` 对 UIColor 和 UIImage 的初始化方法进行了扩展，为了避免冲突，在 Object-C 中添加了 `dm_` 的前缀，在 swift 中，在初始化方法前面添加了一个自定义的枚举 `DMNamespace` 参数。

**UIColor**

```objective-c
NS_ASSUME_NONNULL_BEGIN

@interface UIColor (DarkModeKit)

+ (UIColor *)dm_colorWithLightColor:(UIColor *)lightColor darkColor:(UIColor *)darkColor
NS_SWIFT_UNAVAILABLE("Use init(_:light:dark:) instead.");

#if __swift__
+ (UIColor *)dm_namespace:(DMNamespace)namespace
      colorWithLightColor:(UIColor *)lightColor
                darkColor:(UIColor *)darkColor NS_SWIFT_NAME(init(_:light:dark:));
#endif

@end

NS_ASSUME_NONNULL_END
```

**UIImage**

```objective-c
NS_ASSUME_NONNULL_BEGIN

@interface UIImage (DarkModeKit)

+ (UIImage *)dm_imageWithLightImage:(UIImage *)lightImage darkImage:(UIImage *)darkImage
NS_SWIFT_UNAVAILABLE("Use init(_:light:dark:) instead.");

#if __swift__
+ (UIImage *)dm_namespace:(DMNamespace)namespace
      imageWithLightImage:(UIImage *)lightImage
                darkImage:(UIImage *)darkImage NS_SWIFT_NAME(init(_:light:dark:));
#endif

@end

NS_ASSUME_NONNULL_END
```

在 Object-C 的代码中，通过 `#if __swift__` 来判断编译环境，通过 `NS_SWIFT_NAME(init(_:light:dark:))` 来指定在 swift 中的方面名称。

**注意：这种形式，并没有起到命名空间的作用。在代码中，依然可以定义相同的方法：**

```swift
import FluentDarkModeKit

extension UIColor {
  convenience init(_ name: DMNamespace, light: UIColor, dark: UIColor) {
    self.init(white: 0, alpha: 1.0)
  }
}
```

这样就覆盖了 `FluentDarkModeKit` 框架中的方法。虽然在实际的编程中都不会这样做。







## 总结:

**FluentDarkModeKit**  利用 NSProxy 动态消息转发思想,当切换主题色时候，从 `UIApplication`开始往下遍历到每个 UIView上,执行 `FluentDarkModeKit`的代理 `dmTraitCollectionDidChange` ,重新赋值 View等一系列控件颜色, 赋予的是一个 NSProxy类，类中包含两种UIColor颜色，利用这个动态消息转发，根据当前主题颜色，返回不同颜色 UIColor 做最终的处理结果





## Reference

[1.FluentDarkModeKit 微软的暗黑模式适配框架](https://juejin.cn/post/6844904110395785224)