# iOS UIView刷新与渲染机制



## 1. UIView 与 CALayer 

- UIView 为 CALayer提供内容，专门负责处理触摸等事件，参与响应链
- CALayer基于CoreAnimation, 全权负责显示内容 contents
- 单一原则，设计模式（负责相应的功能）



## 2. 图像渲染流水线

![image-20190312112603570](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-22-024844.png)

图像渲染流程粗粒度地大概分为下面这些步骤：

![在这里插入图片描述](https://sylarimage.oss-cn-shenzhen.aliyuncs.com/uPic/a959d58c9c084ae2a7de5a8df07ae056~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

上述图像渲染流水线中，除了第一部分 Application 阶段，后续主要都由 GPU 负责。



CALayer 是显示的基础：存储 bitmap

CALayer 中的 contents 属性保存了由设备渲染流水线渲染好的位图 bitmap（通常也被称为 backing store），而当设备屏幕进行刷新时，会从 CALayer 中读取生成好的 bitmap，进而呈现到屏幕上



CPU和GPU通过总线连接，CPU中计算出的往往是`bitmap`位图，通过总线由合适的时机传递给GPU，GPU拿到位图后，渲染到帧缓存区`FrameBuffer`,然后由视频控制器根据`Vsync`信号在指定时间之前去帧缓冲区提取内容，显示到屏幕上。

`CPU工作内容:`

1. layout（UI布局，文本计算）
2. display（绘制 drawRect）
3. prepare(图片解码)
4. commit（提交位图）

`GPU工作内容:` 顶点着色，图元装配，光栅化，片段着色，片段处理，最后提交帧缓冲区



## 3. UIView的绘制原理

![image-20190312141642996](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-22-024854.png)



![2251862-1023aed5f7fa5d34.png](https://sylarimage.oss-cn-shenzhen.aliyuncs.com/uPic/b7faae9221b545a19cb2fcb06603922d~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

`[UIView setNeedsDisplay]` 并没有发生当前视图立即绘制工作,打上需要重绘的脏标记，最后是在某个时机完成

`[UIView setLayoutIfNeed]` 立即重新布局视图(下一个Runloop)

`[view layouIfNeeded]` 当前RunLoop休眠前更新

当我们调用UIView的`setNeedsDisplay`的方法时候，会调用`layer`的同名方法，相当于在当前`layer`打上绘制标记，在当前`runloop`将要结束的时候，才会调用CALayer的`display`方法进入到真正的绘制当中。

CALayer的`display`方法中，首先会判断layer的delegate方法`displayLayer：`是否实现，如果代理没有响应这个方法，则进入到系统绘制流程；如果代理响应了这个方法，则进入到异步绘制流程



### 3.1 系统绘制流程

![image-20190312142115333](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-22-024857.png)



Drawrect方法内为何第一行代码总要获取图形的上下文?

**系统绘制的流程** 本质是创建一个 backing storage 的流程.

```objective-c
CGContextRef con = UIGraphicsGetCurrentContext();
```

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2020-03-04-082448.jpg)

每一个UIView都有一个layer，每一个layer都有个content，这个content指向的是一块缓存，叫做backing store
当UIView被绘制时（从 CA::Transaction::commit:以后），CPU执行drawRect，通过context将数据写入backing store
当backing store写完后，通过render server交给GPU去渲染，将backing store中的bitmap数据显示在屏幕上
所以在 drawRect 方法中 要首先获取 context



在CALayer内部，系统会创建一个backingStore（可以理解为CGContextRef，drawRect中取到的currentRef就是这个东西），然后layer回判断是否有delegate，如果没有代理，就调用CALayer的`drawInContext：`方法；如果有代理，则调用layer代理的`drawLayer:inContext:`方法，这一步发生在系统内部，然后在合适的时间给与我们回调一个熟悉的UIView的`drawRect：`方法。也就是在系统内部的绘制之上，允许我们再做一些额外的绘制。最后CALayer把backting store（位图）传给GPU。



### 3.2 异步绘制流程

- UIView 中有一个 CALayer 的属性，负责 UIView 具体内容的显示。
- 具体过程是系统会把 UIView 显示的内容（包括 UILabel 的文字，UIImageView 的图片等）绘制在一张画布上，完成后倒出图片赋值给 CALayer 的 contents 属性，完成显示。

这其中的工作都是在主线程中完成的，这就导致了主线程频繁的处理 UI 绘制的工作，如果要绘制的元素过多，过于频繁，就会造成卡顿。

解决方案使用异步绘制就是：

- 把 UIView 显示的内容（包括 UILabel 的文字，UIImageView 的图片等）绘制生成的 bitmap 在子线程完成。
- 然后在回到主线程把 bitmap 赋值给 view.layer.content 属性。



### 

![image-20190312142425272](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-22-024902.png)

1. 首先在主线程调用 setNeedsdispay 方法
2. 系统会在 runloop 将要结束的时候调用 [CAlayer display] 方法
3. 如果我们的代理实现了dispayLayer 这个方法，会调用 dispayLayer 这个方法。我们可以去子线程里面进行异步绘制。子线程主要做的工作：
   - 创建上下文
   - UI控件的绘制工作
   - 生成对应的图片（bitmap）
4. 主线程可以做其他工作
5. 异步绘制完事之后，回到主线程，把绘制的 bitmap 赋值 view.layer.contents 属性中



![img](https://sylarimage.oss-cn-shenzhen.aliyuncs.com/uPic/3vszdqr4ae.png)





##### **▐**  **是否知道异步绘制？如何进行异步绘制？**

- 基于系统开的口子 [layer.delegate dispayLayer:] 方法。
- 并且实现/遵从了 dispayLayer 这个方法，我们就可以进行异步绘制： 
  - 1）代理负责生产对应的 bitmap 
  - 2）设置 bitmap 作为 layer.contents 属性的值







## 4. View布局与约束时机

一个视图的布局指的是它在屏幕上的的大小和位置。每个 view 都有一个 frame 属性，用来表示在父 view 坐标系中的位置和具体的大小。`UIView` 给你提供了用来通知系统某个 view 布局发生变化的方法，也提供了在 view 布局重新计算后调用的可重写的方法。



### Update Cycle

Update cycle 是当应用完成了你的所有事件处理代码后控制流回到主 RunLoop 时的那个时间点。正是在这个时间点上系统开始更新布局、显示和设置约束。如果你在处理事件的代码中请求修改了一个 view，那么系统就会把这个 view 标记为需要重画（redraw）。在**接下来**的 Update cycle 中，系统就会执行这些 view 上的更改。用户交互和布局更新间的延迟几乎不会被用户察觉到。iOS 应用一般以 60 fps 的速度展示动画，就是说每个更新周期只需要 1/60 秒。这个更新的过程很快，所以用户在和应用交互时感觉不到 UI 中的更新延迟。但是由于在处理事件和对应 view 重画间存在着一个间隔，RunLoop 中的某时刻的 view 更新可能不是你想要的那样。如果你的代码中的某些计算依赖于当下的 view 内容或者是布局，那么就有在过时 view 信息上操作的风险。理解 RunLoop、update cycle 和 `UIView` 中具体的方法可以帮助避免或者可以调试这类问题。下面的图展示出了 update cycle 发生在 RunLoop 的尾部。

![Update Cycle](https://sylarimage.oss-cn-shenzhen.aliyuncs.com/uPic/161d67701eda3640~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.png)



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



如下图，分别为 布局，显示，约束 3个阶段方法；不同方法在不同周期会刷新布局显示出来。



![屏幕快照 2017-10-16 上午12.43.38.png](https://sylarimage.oss-cn-shenzhen.aliyuncs.com/uPic/161d67701eeb4352~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.png)



##### **▐**  **我们调用 [UIView setNeedsDisplay] 方法的时候，不会立马发送对应视图的绘制工作，为什么？**

- 调用 [UIView setNeedsDisplay] 后，
- 然后会调用系统的同名方法 [view.layer setNeedsDisplay] 方法并在当前 view 上面打上一个脏标记
- 当前 Runloop 将要结束的时候才会调用 [CALyer display] 方法，然后进入到视图真正的绘制工作当中。





## 5. View绘制渲染机制和Runloop什么关系



### 底层原理

当在操作 UI 时，，比如修改了frame、调整了UI层级（UIView/CALayer）或者手动设置了setNeedsDisplay:/setNeedsLayout:，这些调整操作会触发transaction commit，这个 `UIView/CALayer` 就被标记为待处理，并被提交到一个全局的容器去。向渲染服务器提交图层树。当这个 Observer 监听了主线程 RunLoop 的即将进入休眠和退出状态，则会遍历所有的UI更新并提交进行实际绘制更新。


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









## 6. UI 卡顿,列表卡顿、掉帧原理

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





## 7. 结合阅读

YYAsyncLayer基于异步绘制:

[YYAsyncLayer 异步绘制原理解析](../SourceCode/YYAsyncLayer.md)



### Reference:

[[译] 揭秘 iOS 布局](https://juejin.im/post/5a951c655188257a804abf94)

[iOS-[渲染原理\]当你被问到下面问题，你能够回答出来么？ 1、app从点击屏幕到完成渲染，中间发生了什么？ 2、当一个 - 掘金 (juejin.cn)](https://juejin.cn/post/7078881864030617607)

[UIView 绘制渲染机制](https://blog.csdn.net/yangyangzhang1990/article/details/52452707)