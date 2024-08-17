# iOS 事件响应链



iOS 事件的主要由：响应连 和 传递链 构成。一般事件先通过传递链，传递下去。响应链，如果上层不能响应，那么一层一层通过响应链找到能响应的`UIResponse`



* 响应连：由最基础的`view`向系统传递，`first view` -> `super view` -> ... -> `view controller` -> `window` -> `Application` -> `AppDelegate`

* 传递链：有系统向最上层`view`传递，`Application` -> `window` -> `root view` -> ... -> `first view`



**注意和事件传递是倆概念!!!!**





## 什么是 iOS 的事件响应链机制？



当事件发生了，必须知道有谁来响应。在iOS中，由响应者链来对事件进行响应。

响应者链是由一个不同对象组成的层次结构，其中的每个对象将依次获得响应事件的机会。当发生事件时，事件首先将被发送到第一响应者，第一响应者基本是事件发生的视图，也就是用户触摸屏幕的地方。事件将沿着响应者链一直向下传递，直到被接受并作出处理。

一般来说，第一响应者是个视图对象或者其子类对象，当其被触摸后事件被交由它处理，如果它不处理，事件就会被传递给它的视图控制器对象 ViewController（如果存在），然后是它的父视图（superview）对象（如果存在），以此类推，直到顶层视图。接下来会沿着顶层视图（top view）到窗口（UIWindow 对象）再到程序（UIApplication 对象）。如果整个过程都没有响应这个事件，该事件就被丢弃。

基本上，在响应者链只要有对象处理事件，事件就停止传递。

```markdown
First Response -> Window -> Application -> nil
```



## 事件传递

**First Response（第一响应者）**，指的是当前接受触摸的响应者对象，是响应者的开端。响应者链和事件分发的使命都是找出第一响应者。



iOS 检测到手指触摸操作（Touch）时，会将其打包成一个 `UIEvent` 对象，并放入当前活动`Application`的事件队列中去。接下来开始自UIApplication往下传递，首先会传递给主window，然后按照view的层级结构一层层往下传递，一直找到最合适的view（发生touch的那个view）来处理事件。查找最合适的view的过程是一个递归的过程，其中涉及到两个重要的方法 `hitTest:withEvent:`和`pointInside:withEvent:`方法寻找出操作初始点所在视图。查找最合适的view的过程是一个递归的过程。



当我们点击屏幕时候的事件传递

从逻辑上来说，探测链是最先发生的机制，当触摸事件发生后，iOS 系统根据 Hit-Testing 来确定触摸事件发生在哪个视图对象上。其中主要用到了两个 UIView 中的方法：

```shell
UIApplication -> UIWindow -> hitTest:withEvent:
```

```objc
- (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event
//返回最终响应的事件
//指定想要响应事件的 View, 比如点击的是 A ，可以指派 B 来响应。
  
- (BOOL)pointInside:(CGPoint)point withEvent:(UIEvent *)event
//判断点击位置是否在当前范围内
//控制响应的范围，扩大 or 缩小。
```

其中UIView不接受事件处理的情况有

```shell
1. hidden ＝ YES,隐藏的视图.
2. userInteractionEnabled = NO,禁止用户操作的视图.
3. alpha <= 0.01, 透明视图
```





![image-20190312105858402](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-22-024827.png)





更好的原理解析如下:

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2020-05-12-143231.jpg)

**Example:**

点击 View D

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2020-05-12-144602.jpg)

```objective-c
- (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event
{
    NSLog(@"进入A_View---hitTest withEvent ---");
    UIView * view = [super hitTest:point withEvent:event];
    NSLog(@"离开A_View--- hitTest withEvent ---hitTestView:%@",view);
    return view;
}

- (BOOL)pointInside:(CGPoint)point withEvent:(nullable UIEvent *)event
{
    NSLog(@"A_view--- pointInside withEvent ---");
    BOOL isInside = [super pointInside:point withEvent:event];
    NSLog(@"A_view--- pointInside withEvent --- isInside:%d",isInside);
    return isInside;
}

- (void)touchesBegan:(NSSet *)touches withEvent:(UIEvent *)event
{
    NSLog(@"A_touchesBegan");
    [super touchesBegan:touches withEvent:event];
}

- (void)touchesMoved:(NSSet<UITouch *> *)touches withEvent:(nullable UIEvent *)event
{
    NSLog(@"A_touchesMoved");
    [super touchesMoved:touches withEvent:event];
}

- (void)touchesEnded:(NSSet<UITouch *> *)touches withEvent:(nullable UIEvent *)event
{
    NSLog(@"A_touchesEnded");
    [super touchesEnded:touches withEvent:event];
}

-(void)touchesCancelled:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event
{
    NSLog(@"A_touchesCancelled");
    [super touchesCancelled:touches withEvent:event];
}



进入A_View---hitTest withEvent ---
A_view--- pointInside withEvent ---
A_view--- pointInside withEvent --- isInside:1
进入C_View---hitTest withEvent ---
C_view---pointInside withEvent ---
C_view---pointInside withEvent --- isInside:1
进入E_View---hitTest withEvent ---
E_view---pointInside withEvent ---
E_view---pointInside withEvent --- isInside:0
离开E_View---hitTest withEvent ---hitTestView:(null)
进入D_View---hitTest withEvent ---
D_view---pointInside withEvent ---
D_view---pointInside withEvent --- isInside:1
离开D_View---hitTest withEvent ---hitTestView:<DView: 0x12dd11e50; frame = (0 37; 240 61); autoresize = RM+BM; layer = <CALayer: 0x283f87b40>>
离开C_View---hitTest withEvent ---hitTestView:<DView: 0x12dd11e50; frame = (0 37; 240 61); autoresize = RM+BM; layer = <CALayer: 0x283f87b40>>
离开A_View--- hitTest withEvent ---hitTestView:<DView: 0x12dd11e50; frame = (0 37; 240 61); autoresize = RM+BM; layer = <CALayer: 0x283f87b40>>

```

如上图,最底层有一个 `AView`, 按顺序添加 `A` 的子View `B C`,  `CView`  按顺序添加 `D E`

如Log, 从底到高传递事件(addSubView顺序倒序遍历 Subviews)

递归执行`hitTest withEvent` 与  `pointInside withEvent` 

如果在 `hitTest ` 后的 `pointInside`检测到该 View 不是触点View,则 `pointInside`返回 NO,` hitTest` 返回nil ,继续遍历 Subviews 倒序下一个,如此反复，直到遍历到最后

**要么至死也没能找到能够响应的对象，最终释放。**



**1. 系统通过 `hitTest:withEvent:` 方法沿视图层级树从底向上（从根视图开始）从后向前（从逻辑上更靠近屏幕的视图开始）进行遍历，最终返回一个适合响应触摸事件的 View。**

**2. 原生触摸事件从 Hit-Testing 返回的 View 开始，沿着响应链从上向下进行传递。**





## 详细触摸事件

**以下的触摸事件更底层的解释:**

### 事件的生命周期

当指尖触碰屏幕的那一刻，一个触摸事件就在系统中生成了。经过IPC进程间通信，事件最终被传递到了合适的应用。在应用内历经峰回路转的奇幻之旅后，最终被释放。大致经过如下图：

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2020-03-01-132050.jpg)

### 系统响应阶段

1. 手指触碰屏幕，屏幕感应到触碰后，将事件交由IOKit处理。
2. IOKit将触摸事件封装成一个IOHIDEvent对象，并通过mach port传递给SpringBoard进程。
3. SpringBoard进程因接收到触摸事件，触发了主线程runloop的source1事件源的回调。



此时SpringBoard会根据当前桌面的状态，判断应该由谁处理此次触摸事件。因为事件发生时，你可能正在桌面上翻页，也可能正在刷微博。若是前者（即前台无APP运行），则触发SpringBoard本身主线程runloop的source0事件源的回调，将事件交由桌面系统去消耗；若是后者（即有app正在前台运行），则将触摸事件通过IPC传递给前台APP进程，接下来的事情便是APP内部对于触摸事件的响应了。



> mach port 进程端口，各进程之间通过它进行通信。
> SpringBoard.app 是一个系统进程，可以理解为桌面系统，可以统一管理和分发系统接收到的触摸事件。



### APP响应阶段

1.APP进程的mach port接受到SpringBoard进程传递来的触摸事件，主线程的runloop被唤醒，触发了source1回调。

2.source1回调又触发了一个source0回调，将接收到的IOHIDEvent对象封装成UIEvent对象，此时APP将正式开始对于触摸事件的响应。

3.source0回调内部将触摸事件添加到UIApplication对象的事件队列中。事件出队后，UIApplication开始一个寻找最佳响应者的过程，这个过程又称hit-testing 。接下来如上面 事件传递 的解释

4.寻找到最佳响应者后，接下来的事情便是事件在响应链中的传递及响应了，关于响应链相关的内容详见[事件的响应及在响应链中的传递]一节。事实上，事件除了被响应者消耗，还能被手势识别器或是target-action模式捕捉并消耗掉。其中涉及对触摸事件的响应优先级

5.触摸事件历经坎坷后要么被某个响应对象捕获后释放，要么致死也没能找到能够响应的对象，最终释放。至此，这个触摸事件的使命就算终结了。runloop若没有其他事件需要处理，也将重归于眠，等待新的事件到来后唤醒。



## Reference

[iOS触摸事件全家桶](https://www.jianshu.com/p/c294d1bd963d)

[深入理解 iOS 事件机制](https://juejin.im/post/5d396ef7518825453b605afa)

[iOS事件处理，看我就够了~](https://juejin.cn/post/6948318786005139493)