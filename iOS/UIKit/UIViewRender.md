# iOS UIView刷新与渲染机制

## CALayer 与 UIView

- UIView 为CALayer提供内容，专门负责处理触摸等事件，参与响应链
- CALayer 全权负责显示内容 contents
- 单一原则，设计模式（负责相应的功能）

## 图像显示原理

![image-20190312112603570](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-22-024844.png)

CPU和GPU通过总线连接，CPU中计算出的往往是`bitmap`位图，通过总线由合适的时机传递给GPU，GPU拿到位图后，渲染到帧缓存区`FrameBuffer`,然后由视频控制器根据`Vsync`信号在指定时间之前去帧缓冲区提取内容，显示到屏幕上。

`CPU工作内容:`

1. layout（UI布局，文本计算）
2. display（绘制 drawRect）
3. prepare(图片解码)
4. commit（提交位图）

`GPU工作内容:` 顶点着色，图元装配，光栅化，片段着色，片段处理，最后提交帧缓冲区

## View绘制渲染机制和Runloop什么关系

例如有以下 `UIView`

```objective-c
@implementation ZYYView
- (void)drawRect:(CGRect)rect {
    CGContextRef con = UIGraphicsGetCurrentContext();
    CGContextAddEllipseInRect(con, CGRectMake(0,0,100,200));
    CGContextSetRGBFillColor(con, 0, 0, 1, 1);
    CGContextFillPath(con);
}
@end
  
  
@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    ZYYView *view = [[ZYYView alloc] init];
    view.backgroundColor = [UIColor whiteColor];
    view.bounds = CGRectMake(0, 0, 100, 100);
    view.center = CGPointMake(100, 100);
    [self.view addSubview:view];
}

@end
```

重写了 `UIView` 的 `DrawRect`方法.展现在屏幕前经历以下堆栈

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2020-03-03-093059.jpg)

### 底层原理

当在操作 UI 时，比如改变了`Frame` 、更新了 `UIView/CALayer` 的层次时，或者手动调用了 `UIView/CALayer` 的 `setNeedsLayout/setNeedsDisplay` 方法后，这个 `UIView/CALayer` 就被标记为待处理，并被提交到一个全局的容器去。
苹果注册了一个 Observer 监听 BeforeWaiting(即将进入休眠) 和 Exit (即将退出Loop) 事件，回调去执行一个很长的函数：
`_ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv()`。这个函数里会遍历所有待处理的 UIView/CAlayer 以执行实际的绘制和调整，并更新 UI 界面。

```objective-c

_ZN2CA11Transaction17observer_callbackEP19__CFRunLoopObservermPv()
QuartzCore:CA::Transaction::observer_callback:
CA::Transaction::commit();
CA::Context::commit_transaction();
CA::Layer::layout_and_display_if_needed();
CA::Layer::layout_if_needed();
[CALayer layoutSublayers];
[UIView layoutSubviews];
CA::Layer::display_if_needed();
[CALayer display];
[UIView drawRect];

```

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2020-03-04-A4.png)

## View布局与约束时机

一个视图的布局指的是它在屏幕上的的大小和位置。每个 view 都有一个 frame 属性，用来表示在父 view 坐标系中的位置和具体的大小。`UIView` 给你提供了用来通知系统某个 view 布局发生变化的方法，也提供了在 view 布局重新计算后调用的可重写的方法。



### 布局:

**layoutSubviews()**

它负责给出当前 view 和每个子 view 的位置和大小。这个方法很开销很大，因为它会在每个子视图上起作用并且调用它们相应的 `layoutSubviews` 方法。系统会在任何它需要重新计算视图的 frame 的时候调用这个方法，所以你应该在需要更新 frame 来重新定位或更改大小时重载它。然而你不应该在代码中显式调用这个方法。相反，有许多可以在 run loop 的不同时间点触发 `layoutSubviews` 调用的机制，这些触发机制比直接调用 `layoutSubviews` 的资源消耗要小得多。




**自动刷新触发器**

有许多事件会自动给视图打上 “update layout” 标记，因此 `layoutSubviews` 会在**下一个周期中（重点！！！）**被调用，而不需要开发者手动操作。这些自动通知系统 view 的布局发生变化的方式有：

- 修改 view 的大小
- 新增 subview
- 用户在 `UIScrollView` 上滚动（`layoutSubviews` 会在 `UIScrollView` 和它的父 view 上被调用）
- 用户旋转设备
- 更新视图的 constraints

这些方式都会告知系统 view 的位置需要被重新计算，继而会自动转化为一个最终的 `layoutSubviews` 调用。当然，也有直接触发 `layoutSubviews` 的方法。



**setNeedsLayout()**

触发 `layoutSubviews` 调用的最省资源的方法就是在你的视图上调用 `setNeedsLaylout` 方法。调用这个方法代表向系统表示视图的布局需要重新计算。`setNeedsLayout` 方法会立刻执行并返回，但在返回前不会真正更新视图。视图会在下一个 update cycle 中更新，就在系统调用视图们的 `layoutSubviews` 以及他们的所有子视图的 `layoutSubviews` 方法的时候。



**layoutIfNeeded()**

`layoutIfNeeded` 是另一个会让 `UIView` 触发 `layoutSubviews` 的方法。 当视图需要更新的时候，与 `setNeedsLayout()` 会让视图在下一周期调用 `layoutSubviews` 更新视图不同，`layoutIfNeeded` 会立即调用 `layoutSubviews` 方法。但是如果你调用了 `layoutIfNeeded` 之后，并且没有任何操作向系统表明需要刷新视图，那么就不会调用 `layoutsubview`。如果你在同一个 run loop 内调用两次 `layoutIfNeeded`，并且两次之间没有更新视图，第二个调用同样不会触发 `layoutSubviews` 方法。

使用 `layoutIfNeeded`，则布局和重绘会立即发生并在函数返回之前完成（除非有正在运行中的动画）。这个方法在你需要依赖新布局，无法等到下一次 update cycle 的时候会比 `setNeedsLayout` 有用



当对希望通过修改 constraint 进行动画时，这个方法特别有用。你需要在 animation block 之前对 self.view 调用 `layoutIfNeeded`，以确保在动画开始之前传播所有的布局更新。在 animation block 中设置新 constrait 后，需要再次调用 `layoutIfNeeded` 来动画到新的状态。

(**注:** Masonry 动画需要这个)



### 显示：

一个视图的显示包含了颜色、文本、图片和 Core Graphics 绘制等视图属性，不包括其本身和子视图的大小和位置。和布局的方法类似，显示也有触发更新的方法，它们由系统在检测到更新时被自动调用，或者我们可以手动调用直接刷新。



**setNeedsDisplay()**

这个方法类似于布局中的 `setNeedsLayout` 。它会给有内容更新的视图设置一个内部的标记，但在视图重绘之前就会返回。然后在下一个 update cycle 中，系统会遍历所有已标标记的视图，并调用它们的 `draw` 方法。

大部分时候，在视图中更新任何 UI 组件都会把相应的视图标记为“dirty”，通过设置视图“内部更新标记”，在下一次 update cycle 中就会重绘，而不需要显式的 `setNeedsDisplay` 调用



### 约束：

**updateConstraints()**

这个方法用来在自动布局中动态改变视图约束。和布局中的 `layoutSubviews()` 方法或者显示中的 `draw` 方法类似，`updateConstraints()` 只应该被重载，**绝不要在代码中显式地调用**。通常你只应该在 `updateConstraints` 方法中实现必须要更新的约束。



**setNeedsUpdateConstraints()**

调用 `setNeedsUpdateConstraints()` 会保证在下一次更新周期中更新约束。它通过标记“update constraints”来触发 `updateConstraints()`。这个方法和 `setNeedsDisplay()` 以及 `setNeedsLayout()` 方法的工作机制类似。



**updateConstraintsIfNeeded()**

对于使用自动布局的视图来说，这个方法与 `layoutIfNeeded` 等价。它会检查 “update constraints”标记（可以被 `setNeedsUpdateConstraints` 或者 `invalidateInstrinsicContentSize`方法自动设置）。如果它认为这些约束需要被更新，它会立即触发 `updateConstraints()` ，**而不会等到 RunLoop 的末尾。**





## UI 卡顿,列表卡顿、掉帧原理

![image-20190312140156990](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-22-024849.png)

iOS的 `mainRunloop`是一个60fps的回调，也就是说每16.7ms(VSync信号时间)会绘制一次屏幕，这个时间段内要完成view的缓冲区创建，view内容的绘制（如果重写了drawRect），这些CPU的工作。然后将这个缓冲区交给GPU渲染，这个过程又包括多个view的拼接(compositing)，纹理的渲染（Texture）等，最终显示在屏幕上。整个过程就是我们上面画的流程图。 因此，如果在16.7ms内完不成这些操作，比如，CPU做了太多的工作，或者view层次过于多，图片过于大，导致GPU压力太大，就会导致“卡”的现象，也就是丢帧.

> 在规定的16.7ms内，在下一个VSync信号到来之前，CPU和GPU并没有共同完成下一帧视频的合成，就会出现掉帧、卡顿。

##### 滑动优化方案思路：

- CPU：
  - 对象的创建、调整、销毁可以放在子线程中去做ASDK；
  - 预排班。布局计算、文本计算等事先放到子线程中去做；
  - 使用轻量级对象，比如CALayer代替UIView
  - 预渲染。文本等异步绘制，图片编解码等。
  - 控制并发线程数量
  - 减少重复计算布局，减少修改frame等
  - autolayout比frame更消耗资源
  - 可以让图片的size跟frame一致
- GPU：
  - 纹理渲染。避免离屏渲染
  - 视图混合。减少视图层级的复杂性，减少透明视图；不透明的opaque设置为YES
  - GPU能处理的最大纹理是4096 * 4096，一旦超过这个尺寸就会调用CPU进行资源处理，所以纹理尽量不要超过这个尺寸

## UIView的绘制原理

![image-20190312141642996](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-22-024854.png)

`[UIView setNeedsDisplay]` 并没有发生当前视图立即绘制工作,打上需要重绘的脏标记，最后是在某个时机完成

`[UIView setLayoutIfNeed]` 立即重新布局视图(下一个Runloop)

`[view layouIfNeeded]` 当前RunLoop休眠前更新

当我们调用UIView的`setNeedsDisplay`的方法时候，会调用`layer`的同名方法，相当于在当前`layer`打上绘制标记，在当前`runloop`将要结束的时候，才会调用CALayer的`display`方法进入到真正的绘制当中。

CALayer的`display`方法中，首先会判断layer的delegate方法`displayLayer：`是否实现，如果代理没有响应这个方法，则进入到系统绘制流程；如果代理响应了这个方法，则进入到异步绘制流程



### 系统绘制流程

![image-20190312142115333](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-22-024857.png)



在CALayer内部，系统会创建一个backingStore（可以理解为CGContextRef，drawRect中取到的currentRef就是这个东西），然后layer回判断是否有delegate，如果没有代理，就调用CALayer的`drawInContext：`方法；如果有代理，则调用layer代理的`drawLayer:inContext:`方法，这一步发生在系统内部，然后在合适的时间给与我们回调一个熟悉的UIView的`drawRect：`方法。也就是在系统内部的绘制之上，允许我们再做一些额外的绘制。最后CALayer把backting store（位图）传给GPU。



![15420320733034](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-07-23-134925.jpg)

1. 首先一个视图由 CPU 进行 Frame 布局，准备视图和图层的层级关系，查询是否有重写 `drawRect:` 或 `drawLayer:inContext: `方法，**注意：如果有重写的话，这里的渲染是会占用CPU进行处理的。**
2. CPU 会将处理视图和图层的层级关系打包，通过 IPC（内部处理通信）通道提交给渲染服务，渲染服务由 OpenGL ES 和 GPU 组成。
3. 渲染服务首先将图层数据交给 OpenGL ES 进行纹理生成和着色。生成前后帧缓存，再根据显示硬件的刷新频率，一般以设备的Vsync信号和CADisplayLink为标准，进行前后帧缓存的切换。
4. 最后，将最终要显示在画面上的后帧缓存交给 GPU，进行采集图片和形状，运行变换，应用文理和混合。最终显示在屏幕上。

> 在iOS中是双缓冲机制，有前帧缓存、后帧缓存，即GPU会预先渲染好一帧放入一个缓冲区内（前帧缓存），让视频控制器读取，当下一帧渲染好后，GPU会直接把视频控制器的指针指向第二个缓冲器（后帧缓存）。当你视频控制器已经读完一帧，准备读下一帧的时候，GPU会等待显示器的VSync信号发出后，前帧缓存和后帧缓存会瞬间切换，后帧缓存会变成新的前帧缓存，同时旧的前帧缓存会变成新的后帧缓存。





### 异步绘制流程

![image-20190312142425272](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-22-024902.png)

layer的delegate如果实现了`displayLayer:`方法，就会进入到异步绘制的流程。在异步绘制的过程中，需要代理来生成对应的bitmap位图文件，并把此bitmap作为layer的contents属性

![image-20190312142514299](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-22-024910.png)



## Drawrect方法内为何第一行代码总要获取图形的上下文

```objective-c
CGContextRef con = UIGraphicsGetCurrentContext();
```

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2020-03-04-082448.jpg)

每一个UIView都有一个layer，每一个layer都有个content，这个content指向的是一块缓存，叫做backing store
当UIView被绘制时（从 CA::Transaction::commit:以后），CPU执行drawRect，通过context将数据写入backing store
当backing store写完后，通过render server交给GPU去渲染，将backing store中的bitmap数据显示在屏幕上
所以在 drawRect 方法中 要首先获取 context





### Reference:

[[译] 揭秘 iOS 布局](https://juejin.im/post/5a951c655188257a804abf94)

[UIView 绘制渲染机制](https://blog.csdn.net/yangyangzhang1990/article/details/52452707)