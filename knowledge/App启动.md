

# App启动过程 

根据Apple官方的《WWDC Optimizing App Startup Time》，iOS应用的启动可分为pre-main阶段和main两个阶段，所以 **App总启动时间 = pre-main耗时 + main耗时**

![](http://okslxr2o0.bkt.clouddn.com/15336131946912.jpg)



一般说来，`pre-main` 阶段的定义为APP**开始启动**到**系统调用main**函数这一段时间；`main`阶段则代表从**main函数入口**到主UI框架的**viewDidAppear**函数调用的这一段时间

App开始启动后，系统首先加载可执行文件（自身App的所有.o文件的集合），然后加载动态链接器dyld，dyld是一个专门用来加载动态链接库的库。 执行从dyld开始，dyld从可执行文件的依赖开始, 递归加载所有的依赖动态链接库。

动态链接库包括：iOS 中用到的所有系统 framework，加载OC runtime方法的libobjc，系统级别的libSystem，例如libdispatch(GCD)和libsystem_blocks (Block)。

**pre-main过程**

![](http://okslxr2o0.bkt.clouddn.com/15326602593912.jpg)


**main过程**

![](http://okslxr2o0.bkt.clouddn.com/15326602955973.jpg)

### 什么是dyld?

动态链接库的加载过程主要由dyld来完成，dyld是苹果的动态链接器。

系统先读取App的可执行文件（Mach-O文件），从里面获得dyld的路径，然后加载dyld，dyld去初始化运行环境，开启缓存策略，加载程序相关依赖库(其中也包含我们的可执行文件)，并对这些库进行链接，最后调用每个依赖库的初始化方法，在这一步，runtime被初始化。当所有依赖库的初始化后，轮到最后一位(程序可执行文件)进行初始化，在这时runtime会对项目中所有类进行类结构初始化，然后调用所有的load方法。最后dyld返回main函数地址，main函数被调用，我们便来到了熟悉的程序入口。

当加载一个 Mach-O 文件 (一个可执行文件或者一个库) 时，动态链接器首先会检查共享缓存看看是否存在其中，如果存在，那么就直接从共享缓存中拿出来使用。每一个进程都把这个共享缓存映射到了自己的地址空间中。这个方法大大优化了 OS X 和 iOS 上程序的启动时间。

### Mach-O 镜像文件

Mach-O 被划分成一些 segement，每个 segement 又被划分成一些 section。segment 的名字都是大写的，且空间大小为页的整数。页的大小跟硬件有关，在 arm64 架构一页是 16KB，其余为 4KB。

section 虽然没有整数倍页大小的限制，但是 section 之间不会有重叠。几乎所有 Mach-O 都包含这三个段（segment）： `__TEXT`，`__DATA`和`__LINKEDIT`。

`__TEXT` 包含 Mach header，被执行的代码和只读常量（如C 字符串）。只读可执行（r-x）。

`__DATA` 包含全局变量，静态变量等。可读写（rw-）。

`__LINKEDIT` 包含了加载程序的『元数据』，比如函数的名称和地址。只读（r–）。

ASLR（Address Space Layout Randomization）：地址空间布局随机化，镜像会在随机的地址上加载。

### 系统使用动态链接有几点好处：

代码共用：很多程序都动态链接了这些 lib，但它们在内存和磁盘中中只有一份。

易于维护：由于被依赖的 lib 是程序执行时才链接的，所以这些 lib 很容易做更新，比如libSystem.dylib 是 libSystem.B.dylib 的替身，哪天想升级直接换成libSystem.C.dylib 然后再替换替身就行了。

减少可执行文件体积：相比静态链接，动态链接在编译时不需要打进去，所以可执行文件的体积要小很多。

### pre-main阶段耗时的影响因素
1. 动态库加载越多，启动越慢。
2. ObjC类越多，函数越多，启动越慢。
3. 可执行文件越大启动越慢。
4. C的constructor函数越多，启动越慢。
5. C++静态对象越多，启动越慢。
6. ObjC的+load越多，启动越慢。

### 整体上pre-main阶段的优化有

1. 减少依赖不必要的库，不管是动态库还是静态库；如果可以的话，把动态库改造成静态库；
如果必须依赖动态库，则把多个非系统的动态库合并成一个动态库；
2. 检查下 framework应当设为optional和required，
如果该framework在当前App支持的所有iOS系统版本都存在，那么就设为required，否则就设为optional，
因为optional会有些额外的检查； 
3. 合并或者删减一些OC类和函数；
关于清理项目中没用到的类，使用工具AppCode代码检查功能，查到当前项目中没有用到的类（也可以用根据linkmap文件来分析，但是准确度不算很高）；
有一个叫做[FUI](https://github.com/dblock/fui)的开源项目能很好的分析出不再使用的类，准确率非常高，唯一的问题是它处理不了动态库和静态库里提供的类，也处理不了C++的类模板。
4. 删减一些无用的静态变量，
5. 删减没有被调用到或者已经废弃的方法，
方法见http://stackoverflow.com/questions/35233564/how-to-find-unused-code-in-xcode-7
和https://developer.Apple.com/library/ios/documentation/ToolsLanguages/Conceptual/Xcode_Overview/CheckingCodeCoverage.html。
6. 将不必须在+load方法中做的事情延迟到+initialize中，尽量不要用C++虚函数(创建虚函数表有开销)
7. 类和方法名不要太长：iOS每个类和方法名都在__cstring段里都存了相应的字符串值，所以类和方法名的长短也是对可执行文件大小是有影响的；
因还是object-c的动态特性，因为需要通过类/方法名反射找到这个类/方法进行调用，object-c对象模型会把类/方法名字符串都保存下来；
8. ⑧用dispatch_once()代替所有的 attribute((constructor)) 函数、C++静态对象初始化、ObjC的+load函数；
9. 在设计师可接受的范围内压缩图片的大小，会有意外收获。
压缩图片为什么能加快启动速度呢？因为启动的时候大大小小的图片加载个十来二十个是很正常的，
图片小了，IO操作量就小了，启动当然就会快了，比较靠谱的压缩算法是TinyPNG。

### main阶段


1. 减少启动初始化的流程，能懒加载的就懒加载，能放后台初始化的就放后台，能够延时初始化的就延时，不要卡主线程的启动时间，已经下线的业务直接删掉； 
2. 优化代码逻辑，去除一些非必要的逻辑和代码，减少每个流程所消耗的时间； 
3. 启动阶段使用多线程来进行初始化，把CPU的性能尽量发挥出来； 
4. 使用纯代码而不是xib或者storyboard来进行UI框架的搭建，尤其是主UI框架比如TabBarController这种，尽量避免使用xib和storyboard，因为xib和storyboard也还是要解析成代码来渲染页面，多了一些步骤； 


### Note:

#### 1
* 静态库 常见的是 .a
* 动态库常见的是 .dll(windows) .dylib(mac) so(linux)
* framework(in Apple): Framework 是 Cocoa/Cocoa Touch 程序中使用的一种资源打包方式，可以将代码文件、头文件、资源文件、说明文档等集中在一起，方便开发者使用。也就是说我们的 framework ，**其实是资源打包的方式，和静态库动态库的本质是没有关系的**

#### 2
* 静态库: 链接时会被完整的复制到可执行文件中，所以如果两个程序都用了某个静态库，那么每个二进制可执行文件里面其实都含有这份静态库的代码
* 动态库: 链接时不复制，在程序启动后用 dyld 加载，然后再决议符号，所以理论上动态库只用存在一份，好多个程序都可以动态链接到这个动态库上面，达到了节省内存(不是磁盘是内存中只有一份动态库)，还有另外一个好处，由于动态库并不绑定到可执行程序上，所以我们想升级这个动态库就很容易，windows 和 linux 上面一般插件和模块机制都是这样实现的。
* Embedded Framework，这种动态库允许APP 和 APP Extension共享代码，但是这份动态库的生命被限定在一个 APP 进程内。简单点可以理解为 **被阉割的动态库**。

#### 3.
简单归纳:
1、main之前的加载过程

* dyld 开始将程序二进制文件初始化
* 交由ImageLoader 读取 image，其中包含了我们的类，方法等各种符号（Class、Protocol 、Selector、 IMP）
* 由于runtime 向dyld 绑定了回调，当image加载到内存后，dyld会通知runtime进行处理
* runtime 接手后调用map_images做解析和处理
* 接下来load_images 中调用call_load_methods方法，遍历所有加载进来的Class，按继承层次依次调用Class的+load和其他Category的+load方法
* 至此 所有的信息都被加载到内存中
* 最后dyld调用真正的main函数


### Reference
1. [iOS启动时间优化](http://www.zoomfeng.com/blog/launch-time.html)
2. [iOS启动时间](http://lingyuncxb.com/2018/01/30/iOS%E5%90%AF%E5%8A%A8%E4%BC%98%E5%8C%96/)

