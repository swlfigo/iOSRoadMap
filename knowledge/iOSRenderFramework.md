# iOS 渲染框架

#  `UIKit`

`UIKit`自身并不具备在屏幕成像的能力，其主要负责对用户操作事件的响应（`UIView`继承自`UIResponder`），事件响应的传递大体是经过逐层的视图树遍历实现的。

# `Core Animation`

`Core Animation`是一个复合引擎，其职责是尽可能快地**组合屏幕上不同的可视内容，这些可视内容可被分解成独立的图层（即CALayer）**，这些图层会被存储在一个叫做图层树的体系之中。

# `Core Graphics` 

`Core Graphics`基于`Quartz`高级绘图引擎，主要用于**运行时绘制图像**。开发者可以使用此框架来处理基于路径的绘图，转换，颜色管理，离屏渲染，图案，渐变和阴影，图像数据管理，图像创建和图像遮罩以及PDF文档创建，

开发者需要在**运行时创建图像**时，可以使用`Core Graphics`去绘制。

# `Core Image`

`Core Image`与`Core Graphics`恰恰相反，`Core Graphics`用于在**运行时创建图像**，而`Core Image`用于**处理运行前创建的图像**。

# `OpenglES`







