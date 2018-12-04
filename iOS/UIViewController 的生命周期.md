### 1. 简述下 UIViewController 的生命周期

ViewController的生命周期中各方法扫行须序如下：

alloc init—>

loadView—> 尽管不直接调用该方法，如多手动创建自己的视图，那么应该覆盖这个方法并将它们赋值给试图控制器的 view 属性。

viewDidLoad—> 只有在视图控制器将其视图载入到内存之后才调用该方法，这是执行任何其他初始化操作的入口。

viewWillAppear—> 当试图将要添加到窗口中并且还不可见的时候或者上层视图移出图层后本视图变成顶级视图时调用该方法，用于执行诸如改变视图方向等的操作。实现该方法时确保调用 [super viewWillAppear:]

viewDidAppear—> 当视图添加到窗口中以后或者上层视图移出图层后本视图变成顶级视图时调用，用于放置那些需要在视图显示后执行的代码。确保调用 [super viewDidAppear:]

viewWillDisappear—>
viewDidDisappear—>
viewWillUnload->
viewDidUnload—>
dealloc->
