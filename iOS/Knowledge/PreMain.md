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



具体查看->   [Dyld](./Dyld.md)



## 总结：

#### main()函数调用之前，其实是做了很多准备工作，主要是dyld这个动态链接器在负责，核心流程如下:

##### 1. 程序执行从_dyld_star开始

- 1.1. 读取macho文件信息，设置虚拟地址偏移量，用于重定向。
- 1.2. 调用dyld::_main方法进入macho文件的主程序。

##### 2. 配置一些环境变量

- 2.1. 设置的环境变量方便我们打印出更多的信息。
- 2.1. 调用getHostInfo()来获取machO头部获取当前运行架构的信息。

##### 3. 实例化主程序,即macho可执行文件。

##### 4. 加载共享缓存库。

##### 5. 插入动态缓存库。

##### 6. 链接主程序。

##### 7. 初始化函数。

- 7.1. 经过一系列的初始化函数最终调用notifSingle函数。
- 7.2. 此回调是被运行时_objc_init初始化时赋值的一个函数load_images
- 7.3. load_images里面执行call_load_methods函数，循环调用所用类以及分类的load方法。
- 7.4. doModInitFunctions函数，内部会调用全局C++对象的构造函数，即_ _ attribute_ _((constructor))这样的函数。

##### 8. 返回主程序的入口函数，开始进入主程序的main（）函数。



## Reference

[1. iOS App从点击到启动](https://juejin.cn/post/6844903783160348685)

[2. 一步一步带你揭开main函数之前的面纱](https://juejin.cn/post/6844903783160348685)