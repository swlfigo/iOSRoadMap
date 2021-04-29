# App Launch Optimize

## 基本概念

### 启动的定义

启动有两种定义：

- 广义：点击图标到首页数据加载完毕
- 狭义：点击图标到 Launch Image 完全消失第一帧

这是从用户感知维度定义启动，那么代码上如何定义启动呢？Apple 在 MetricKit 中给出了官方计算方式：

- 起点：进程创建的时间
- 终点：第一个`CA::Transaction::commit()`

> Tips：`CATransaction` 是 Core Animation 提供的一种事务机制，把一组 UI 上的修改打包，一起发给 Render Server 渲染。

### 启动的种类

根据场景的不同，启动可以分为三种：冷启动，热启动和回前台。

- 冷启动：系统里没有任何进程的缓存信息，典型的是重启手机后直接启动 App
- 热启动：如果把 App 进程杀了，然后立刻重新启动，这次启动就是热启动，因为进程缓存还在
- 回前台：大多数时候不会被定义为启动，因为此时 App 仍然活着，只不过处于 suspended 状态



### Mach-O

Mach-O 是 iOS 可执行文件的格式，典型的 Mach-O 是主二进制和动态库。Mach-O 可以分为三部分：

- Header
- Load Commands
- Data



![图片](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-04-29-094446.jpg)



Header 的最开始是 Magic Number，表示这是一个 Mach-O 文件，除此之外还包含一些 Flags，这些 flags 会影响 Mach-O 的解析。

**Load Commands 存储 Mach-O 的布局信息**，比如 Segment command 和 Data 中的 Segment/Section 是一一对应的。除了布局信息之外，还包含了依赖的动态库等启动 App 需要的信息。

**Data 部分包含了实际的代码和数据**，Data 被分割成很多个 Segment，每个 Segment 又被划分成很多个 Section，分别存放不同类型的数据。

标准的三个 Segment 是 TEXT，DATA，LINKEDIT，也支持自定义：

- **TEXT**，代码段，只读可执行，存储函数的二进制代码(__text)，常量字符串(__cstring)，Objective C 的类/方法名等信息
- **DATA**，数据段，读写，存储 Objective C 的字符串(__cfstring)，以及运行时的元数据：class/protocol/method…
- **LINKEDIT**，启动 App 需要的信息，如 bind & rebase 的地址，代码签名，符号表…



### dyld

dyld 是启动的辅助程序，是 in-process 的，即启动的时候会把 dyld 加载到进程的地址空间里，然后把后续的启动过程交给 dyld。dyld 主要有两个版本：dyld2 和 dyld3。

dyld2 是从 iOS 3.1 引入，一直持续到 iOS 12。dyld2 有个比较大的优化是 **dyld shared cache**[1]，什么是 shared cache 呢？

- shared cache 就是把系统库(UIKit 等)合成一个大的文件，提高加载性能的缓存文件。

iOS 13 开始 Apple 对三方 App 启用了 dyld3，dyld3 的最重要的特性就是启动闭包，闭包里包含了启动所需要的缓存信息，从而提高启动速度。

### 虚拟内存

内存可以分为虚拟内存和物理内存，其中物理内存是实际占用的内存，虚拟内存是在物理内存之上建立的一层逻辑地址，保证内存访问安全的同时为应用提供了连续的地址空间。

物理内存和虚拟内存以页为单位映射，但这个映射关系不是一一对应的：一页物理内存可能对应多页虚拟内存；一页虚拟内存也可能不占用物理内存。

![图片](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-04-29-100734.jpg)**iPhone 6s 开始，物理内存的 Page 大小是 16K，6 和之前的设备都是 4K，这是 iPhone 6 相比 6s 启动速度断崖式下降的原因之一**。



### mmap

mmap 的全称是 memory map，是一种内存映射技术，可以把文件映射到虚拟内存的地址空间里，这样就可以像直接操作内存那样来读写文件。**当读取虚拟内存，其对应的文件内容在物理内存中不存在的时候，会触发一个事件：File Backed Page In，把对应的文件内容读入物理内存**。

启动的时候，Mach-O 就是通过 mmap 映射到虚拟内存里的(如下图)。下图中部分页被标记为 zero fill，是因为全局变量的初始值往往都是 0，那么这些 0 就没必要存储在二进制里，增加文件大小。操作系统会识别出这些页，在 Page In 之后对其置为 0，这个行为叫做 zero fill。

![图片](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-04-29-100752.jpg)

### Page In

启动的路径上会触发很多次 Page In，其实也比较容易理解，因为启动的会读写二进制中的很多内容。**Page In 会占去启动耗时的很大一部分**，我们来看看单个 Page In 的过程：

![图片](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-04-29-100903.png)

- MMU 找到空闲的物理内存页面
- 触发磁盘 IO，把数据读入物理内存
- 如果是 TEXT 段的页，要进行解密
- 对解密后的页，进行签名验证

其中**解密是大头，IO 其次**。为什么要解密呢？

因为 iTunes Connect 会对上传 Mach-O 的 TEXT 段进行加密，防止 IPA 下载下来就直接可以看到代码。这也就是为什么逆向里会有个概念叫做“砸壳”，砸的就是这一层 TEXT 段加密。**iOS 13 对这个过程进行了优化，Page In 的时候不需要解密了**。



### 二进制重排

既然 Page In 耗时，有没有什么办法优化呢？

启动具有**局部性特征**，即只有少部分函数在启动的时候用到，这些函数在二进制中的分布是零散的，所以 Page In 读入的数据利用率并不高。如果我们可以把启动用到的函数排列到二进制的连续区间，那么就可以减少 Page In 的次数，从而优化启动时间：

以下图为例，方法 1 和方法 3 是启动的时候用到的，为了执行对应的代码，就需要两次 Page In。假如我们把方法 1 和 3 排列到一起，那么只需要一次 Page In，从而提升启动速度。

![图片](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-04-29-100931.jpg)链接器 ld 有个参数-order_file 支持按照符号的方式排列二进制。获取启动时候用到的符号的有很多种方式，感兴趣的同学可以看看抖音之前的文章：[**基于二进制文件重排的解决方案 APP 启动速度提升超 15%**](http://mp.weixin.qq.com/s?__biz=MzI1MzYzMjE0MQ==&mid=2247485101&idx=1&sn=abbbb6da1aba37a04047fc210363bcc9&chksm=e9d0cd4fdea7445989cf26623a16fc8ce2876bf3bda95a5532bb0e5e5b1420765653df0b94d1&scene=21#wechat_redirect)。



## IPA 构建

### pipeline

既然要构建，那么必然会有一些地方去定义如何构建，对应 Xcode 中的两个配置项：

- **Build Phase：以 Target 为维度定义了构建的流程**。可以在 Build Phase 中插入脚本，来做一些定制化的构建，比如 CocoaPod 的拷贝资源就是通过脚本的方式完成的。
- **Build Settings：配置编译和链接相关的参数**。特别要提到的是 other link flags 和 other c flags，因为编译和链接的参数非常多，有些需要手动在这里配置。很多项目用的 CocoaPod 做的组件化，这时候编译选项在对应的.xcconfig 文件里。

以单 Target 为例，我们来看下构建流程：![图片](https://mmbiz.qpic.cn/mmbiz_png/5EcwYhllQOhKWibuGWDU1lBFc20nxt254glw8jJgPqxq5o7ZrkAZVHianfbichVEDwSfkf9iaRq3dATHgVtz0iawo0Q/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

- 源文件(.m/.c/.swift 等)是单独编译的，输出对应的目标文件(.o)
- 目标文件和静态库/动态库一起，链接出最后的 Mach-O
- Mach-O 会被裁剪，去掉一些不必要的信息
- 资源文件如 storyboard，asset 也会编译，编译后加载速度会变快
- Mach-O 和资源文件一起，打包出最后的.app
- 对.app 签名，防篡改

### 编译

编译器可以分为两大部分：前端和后端，二者以 IR（中间代码）作为媒介。这样前后端分离，使得前后端可以独立的变化，互不影响。C 语言家族的前端是 clang，swift 的前端是 swiftc，二者的后端都是 llvm。

- 前端负责预处理，词法语法分析，生成 IR
- 后端基于 IR 做优化，生成机器码

![图片](https://mmbiz.qpic.cn/mmbiz_png/5EcwYhllQOhKWibuGWDU1lBFc20nxt254icA9NhyYqAjNHAsZPyxNyCE7zoABd4petfF4nf8urvyjJQYYOHMHiceg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)那么如何利用编译优化启动速度呢？

**代码数量会影响启动速度，为了提升启动速度，我们可以把一些无用代码下掉**。那怎么统计哪些代码没有用到呢？可以利用 LLVM 插桩来实现。LLVM 的代码优化流程是一个一个 Pass，由于 LLVM 是开源的，我们可以添加一个自定义的 Pass，在函数的头部插入一些代码，这些代码会记录这个函数被调用了，然后把统计到的数据上传分析，就可以知道哪些代码是用不到的了 。

Facebook 给 LLVM 提的 **order_file**[2]的 feature 就是实现了类似的插桩。

### 链接

经过编译后，我们有很多个目标文件，接着这些目标文件会和静态库，动态库一起，链接出一个 Mach-O。链接的过程并不产生新的代码，只会做一些移动和补丁。![图片](https://mmbiz.qpic.cn/mmbiz_png/5EcwYhllQOhKWibuGWDU1lBFc20nxt254FlTjFwZibehWfy900icNRR7tEopibAn56mqtLgTiaBTehl6sPibe8UhnHzQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

- tbd 的全称是 text-based stub library，是因为链接的过程中只需要符号就可以了，所以 Xcode 6 开始，像 UIKit 等系统库就不提供完整的 Mach-O，而是提供一个只包含符号等信息的 tbd 文件。