# main 函数之前发生了什么

大体分为如下步骤：

(1) 系统为程序启动做好准备

(2) 系统将控制权交给 Dyld，Dyld 会负责后续的工作

(3) Dyld 加载程序所需的动态库

(3) Dyld 对程序进行 rebase 以及 bind 操作

(4) Objc SetUp

(5) 运行初始化函数

(6) 执行程序的 main 函数

需要注意的是，dyld2和dyld3的加载方式略有不同。dyld2是纯粹的in-process，也就是在程序进程内执行的，也就意味着只有当应用程序被启动的时候，dyld2才能开始执行任务。dyld3则是部分out-of-process，部分in-process。

dyld2的过程是：加载dyld到App进程，加载动态库（包括所依赖的所有动态库），Rebase，Bind，初始化Objective C Runtime和其它的初始化代码。

dyld3的out-of-process会做如下事情：分析Mach-o Headers，分析依赖的动态库，查找需要Rebase & Bind之类的符号，把上述结果写入缓存。这样，在应用启动的时候，就可以直接从缓存中读取数据，加快加载速度。



#### Dyld

Dyld 是 iOS 系统的动态链接器， 在dyldStartup.s 文件中有个名为 __dyld_start 的方法，它会去调用 dyldbootstrap::start() 方法，然后进一步调用 dyld::_main() 方法，里面包含 App 的整个启动流程，该函数最终返回应用程序 main 函数的地址，最后 Dyld 会去调用它。

之后会去加载可执行文件，二进制文件常被称为 image，包括可执行文件、动态库等，ImageLoader 的作用就是将二进制文件加载进内存。dyld::_main() 方法在设置好运行环境后，会调用instantiateFromLoadedImage 函数将可执行文件加载进内存中，加载过程分为三步：



```undefined
合法性检查。主要是检查可执行文件是否合法，是否能在当前的 CPU 架构下运行。
选择 ImageLoader 加载可执行文件。系统会去判断可执行文件的类型，选择相应的 ImageLoader 将其加载进内存空间中。
注册 image 信息。可执行文件加载完成后，系统会调用 addImage 函数将其管理起来，并更新内存分布信息。
```

以上三步完成后，Dyld 会调用 link 函数开始之后的处理流程。

##### 



作者：一个人在路上走下去
链接：https://www.jianshu.com/p/231b1cebf477
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

作者：一个人在路上走下去
链接：https://www.jianshu.com/p/231b1cebf477
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

https://www.jianshu.com/p/71c38b56c61a

https://juejin.cn/post/6844903783160348685