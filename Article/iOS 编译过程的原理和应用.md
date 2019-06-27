>  原文地址 https://mp.weixin.qq.com/s/32W4orJWvRkKXwSCzjkxGA

![](https://mmbiz.qpic.cn/mmbiz_png/3QD99b9DjVHWud0atIQxpVMH4JtmOTHCGT0UicQtdU5d6VryvickA8HUTBFVLcDtlpmpU10zXNt5xRzTTcP5hFFA/640?wx_fmt=png)

<section>

## 前言

一般可以将编程语言分为两种，编译语言和直译式语言。

像 C++,Objective C 都是编译语言。编译语言在执行的时候，必须先通过编译器生成机器码，机器码可以直接在 CPU 上执行，所以执行效率较高。

像 JavaScript,Python 都是直译式语言。直译式语言不需要经过编译的过程，而是在执行的时候通过一个中间的解释器将代码解释为 CPU 可以执行的代码。所以，较编译语言来说，直译式语言效率低一些，但是编写的更灵活，也就是为啥 JS 大法好。

iOS 开发目前的常用语言是：Objective 和 Swift。二者都是编译语言，换句话说都是需要编译才能执行的。二者的编译都是依赖于 Clang + LLVM. 篇幅限制，本文只关注 Objective C，因为原理上大同小异。

可能会有同学想问，我不懂编译的过程，写代码也没问题啊？这点我是不否定的。但是，充分理解了编译的过程，会对你的开发大有帮助。本文的最后，会以以下几个例子，来讲解如何合理利用 XCode 和编译

*    **attribute**

*   Clang 警告处理

*   预处理

*   插入编译期脚本

*   提高项目编译速度

对于不想看我啰里八嗦讲一大堆原理的同学，可以直接跳到本文的最后一个章节。

## iOS 编译

Objective C 采用 Clang(swift 采用 swift) 作为编译器前端，LLVM(Low level vritual machine) 作为编译器后端。

简单的编译过程如图

![](https://mmbiz.qpic.cn/mmbiz_png/3QD99b9DjVHWud0atIQxpVMH4JtmOTHCZNcjOibVzMQYcbnLriaOTezeVVT7QIrYfFdxhB6HfLzWucCibrnmUmXaA/640?wx_fmt=png)

### 编译器前端

> 编译器前端的任务是进行：语法分析，语义分析，生成中间代码 (intermediate representation)。在这个过程中，会进行类型检查，如果发现错误或者警告会标注出来在哪一行。

![](https://mmbiz.qpic.cn/mmbiz_png/3QD99b9DjVHWud0atIQxpVMH4JtmOTHCI8vTyvibC5SUh0Mun4OXNict5NtBvicJFtwhroOXwsxpsl2arn4t66ZsA/640?wx_fmt=png)

### 编译器后端

> 编译器后端会进行机器无关的代码优化，生成机器语言，并且进行机器相关的代码优化。iOS 的编译过程，后端的处理如下

*   LVVM 优化器会进行 BitCode 的生成，链接期优化等等。

![](https://mmbiz.qpic.cn/mmbiz_png/3QD99b9DjVHWud0atIQxpVMH4JtmOTHCShj6r88lM201oEf7WqFcMRVZkoGZicqibywHHGy787xmrE1EdJJ8p7Ag/640?wx_fmt=png)

LLVM 机器码生成器会针对不同的架构，比如 arm64 等生成不同的机器码。

![](https://mmbiz.qpic.cn/mmbiz_png/3QD99b9DjVHWud0atIQxpVMH4JtmOTHC7RDgjR2IryKib5HNISnLVmcnATmfSdroZbnLicxlL6Uu3J9shdPQMPaQ/640?wx_fmt=png)

## 执行一次 XCode build 的流程

当你在 XCode 中，选择 build 的时候（快捷键 command+B），会执行如下过程

*   编译信息写入辅助文件，创建编译后的文件架构 (name.app)

*   处理文件打包信息，例如在 debug 环境下

<pre>Entitlements:
{
    "application-identifier" = "app的bundleid";
    "aps-environment" = development;
}

</pre>

*   执行 CocoaPod 编译前脚本

*   例如对于使用 CocoaPod 的工程会执行 CheckPods Manifest.lock

*   编译各个. m 文件，使用 CompileC 和 clang 命令。

<pre>CompileC ClassName.o ClassName.m normal x86_64 objective-c com.apple.compilers.llvm.clang.1_0.compiler
export LANG=en_US.US-ASCII
export PATH="..."
clang -x objective-c -arch x86_64 -fmessage-length=0 -fobjc-arc... -Wno-missing-field-initializers ... -DDEBUG=1 ... -isysroot iPhoneSimulator10.1.sdk -fasm-blocks ... -I 上文提到的文件 -F 所需要的Framework  -iquote 所需要的Framework  ... -c ClassName.c -o ClassName.c
</pre>

通过这个编译的命令，我们可以看到

<pre>clang是实际的编译命令
-x         objective-c 指定了编译的语言
-arch     x86_64制定了编译的架构，类似还有arm7等
-fobjc-arc 一些列-f开头的，指定了采用arc等信息。这个也就是为什么你可以对单独的一个.m文件采用非ARC编程。
-Wno-missing-field-initializers 一系列以-W开头的，指的是编译的警告选项，通过这些你可以定制化编译选项
-DDEBUG=1 一些列-D开头的，指的是预编译宏，通过这些宏可以实现条件编译
-iPhoneSimulator10.1.sdk 制定了编译采用的iOS SDK版本
-I 把编译信息写入指定的辅助文件
-F 链接所需要的Framework
-c ClassName.c 编译文件
-o ClassName.o 编译产物

</pre>

*   链接需要的 Framework，例如 Foundation.framework,AFNetworking.framework,ALiPay.fframework

*   编译 xib 文件

*   拷贝 xib，图片等资源文件到结果目录

*   编译 ImageAssets

*   处理 info.plist

*   执行 CocoaPod 脚本

*   拷贝 Swift 标准库

*   创建. app 文件和对其签名

## IPA 包的内容

例如，我们通过 iTunes Store 下载微信，然后获得 ipa 安装包，然后实际看看其安装包的内容。

![](https://mmbiz.qpic.cn/mmbiz_png/3QD99b9DjVHWud0atIQxpVMH4JtmOTHCaRTxr41uQj1Q2NSlh4ZbSfrczWSq3zNib6V35xNAlJfLibAfPvXIwDfA/640?wx_fmt=png)

*   右键 ipa，重命名为. zip

*   双击 zip 文件，解压缩后会得到一个文件夹。所以，ipa 包就是一个普通的压缩包。

![](https://mmbiz.qpic.cn/mmbiz_png/3QD99b9DjVHWud0atIQxpVMH4JtmOTHC4SMfNINjQjGZ2y7UtMicGrj5GQKSGaX8hFAOYLF1JibeWYS8IdaATA2A/640?wx_fmt=png)

*   右键图中的`WeChat`，选择显示包内容，然后就能够看到实际的 ipa 包内容了。

## 二进制文件的内容

通过 XCode 的 Link Map File，我们可以窥探二进制文件中布局。 在 XCode -> Build Settings -> 搜索 map -> 开启 Write Link Map File

![](https://mmbiz.qpic.cn/mmbiz_png/3QD99b9DjVHWud0atIQxpVMH4JtmOTHCT5j29CKY49GcZRXlnJNNxyRfYgRb30D43fYKtWBbNa9M2MXqjbkk7g/640?wx_fmt=png)

开启后，在编译，我们可以在对应的 Debug/Release 目录下看到对应的 link map 的 text 文件。 默认的目录在

<pre>~/Library/Developer/Xcode/DerivedData/<TARGET-NAME>-对应ID/Build/Intermediates/<TARGET-NAME>.build/Debug-iphoneos/<TARGET-NAME>.build/
</pre>

例如，我的 TargetName 是 EPlusPan4Phone，目录如下

<pre>/Users/huangwenchen/Library/Developer/Xcode/DerivedData/EPlusPan4Phone-eznmxzawtlhpmadnbyhafnpqpizo/Build/Intermediates/EPlusPan4Phone.build/Debug-iphonesimulator/EPlusPan4Phone.build
</pre>

这个映射文件的主要包含以下部分：

### Object files

这个部分包括的内容

*   .o 文文件，也就是上文提到的. m 文件编译后的结果。

*   .a 文件

*   需要 link 的 framework

*   *

> #### ！ Arch: x86_64 #Object files: [0] linker synthesized [1] /EPlusPan4Phone.build/EPlusPan4Phone.app.xcent [2]/EPlusPan4Phone.build/Objects-normal/x86_64/ULWBigResponseButton.o … [1175]/UMSocial_Sdk_4.4/libUMSocial_Sdk_4.4.a(UMSocialJob.o) [1188]/iPhoneSimulator10.1.sdk/System/Library/Frameworks//Foundation.framework/Foundation

这个区域的存储内容比较简单：前面是文件的编号，后面是文件的路径。文件的编号在后续会用到

## Sections

这个区域提供了各个段（Segment）和节（Section）在可执行文件中的位置和大小。这个区域完整的描述克可执行文件中的全部内容。

其中，段分为两种

*   __TEXT 代码段

*   __DATA 数据段

例如，之前写的一个 App，Sections 区域如下，可以看到，代码段的

__text 节的地址是 0x1000021B0，大小是 0x0077EBC3，而二者相加的下一个位置正好是__stubs 的位置 0x100780D74。

<pre># Sections:
# 位置       大小        段       节
# Address    Size        Segment Section
0x1000021B0    0x0077EBC3  __TEXT  __text //代码
0x100780D74    0x00000FD8  __TEXT  __stubs
0x100781D4C    0x00001A50  __TEXT  __stub_helper
0x1007837A0    0x0001AD78  __TEXT  __const //常量
0x10079E518    0x00041EF7  __TEXT  __objc_methname //OC 方法名
0x1007E040F    0x00006E34  __TEXT  __objc_classname //OC 类名
0x1007E7243    0x00010498  __TEXT  __objc_methtype  //OC 方法类型
0x1007F76DC    0x0000E760  __TEXT  __gcc_except_tab 
0x100805E40    0x00071693  __TEXT  __cstring  //字符串
0x1008774D4    0x00004A9A  __TEXT  __ustring  
0x10087BF6E    0x00000149  __TEXT  __entitlements 
0x10087C0B8    0x0000D56C  __TEXT  __unwind_info 
0x100889628    0x000129C0  __TEXT  __eh_frame
0x10089C000    0x00000010  __DATA  __nl_symbol_ptr
0x10089C010    0x000012C8  __DATA  __got
0x10089D2D8    0x00001520  __DATA  __la_symbol_ptr
0x10089E7F8    0x00000038  __DATA  __mod_init_func
0x10089E840    0x0003E140  __DATA  __const //常量
0x1008DC980    0x0002D840  __DATA  __cfstring
0x10090A1C0    0x000022D8  __DATA  __objc_classlist // OC 方法列表
0x10090C498    0x00000010  __DATA  __objc_nlclslist 
0x10090C4A8    0x00000218  __DATA  __objc_catlist
0x10090C6C0    0x00000008  __DATA  __objc_nlcatlist
0x10090C6C8    0x00000510  __DATA  __objc_protolist // OC协议列表
0x10090CBD8    0x00000008  __DATA  __objc_imageinfo
0x10090CBE0    0x00129280  __DATA  __objc_const // OC 常量
0x100A35E60    0x00010908  __DATA  __objc_selrefs
0x100A46768    0x00000038  __DATA  __objc_protorefs 
0x100A467A0    0x000020E8  __DATA  __objc_classrefs 
0x100A48888    0x000019C0  __DATA  __objc_superrefs // OC 父类引用
0x100A4A248    0x0000A500  __DATA  __objc_ivar // OC iar
0x100A54748    0x00015CC0  __DATA  __objc_data
0x100A6A420    0x00007A30  __DATA  __data
0x100A71E60    0x0005AF70  __DATA  __bss
0x100ACCDE0    0x00053A4C  __DATA  __common

</pre>

## Symbols

Section 部分将二进制文件进行了一级划分。而，Symbols 对 Section 中的各个段进行了二级划分， 例如，对于__TEXT __text, 表示代码段中的代码内容。

<pre>0x1000021B0    0x0077EBC3  __TEXT  __text //代码
</pre>

而对应的 Symbols，起始地址也是 0x1000021B0 。其中，文件编号和上文的编号对应

<pre>[2]/EPlusPan4Phone.build/Objects-normal/x86_64/ULWBigResponseButton.o
</pre>

具体内容如下

<pre># Symbols:
  地址     大小          文件编号    方法名
# Address    Size        File       Name
0x1000021B0    0x00000109  [  2]     -[ULWBigResponseButton pointInside:withEvent:]
0x1000022C0    0x00000080  [  3]     -[ULWCategoryController liveAPI]
0x100002340    0x00000080  [  3]     -[ULWCategoryController categories]
....

</pre>

到这里，我们知道 OC 的方法是如何存储的，我们再来看看 ivar 是如何存储的。 首先找到数据栈中__DATA __objc_ivar

<pre>0x100A4A248    0x0000A500  __DATA  __objc_ivar
</pre>

然后，搜索这个地址 0x100A4A248，就能找到 ivar 的存储区域。

<pre>0x100A4A248    0x00000008  [  3] _OBJC_IVAR_$_ULWCategoryController._liveAPI
</pre>

值得一提的是，对于 String，会显式的存储到数据段中，例如,

<pre>0x1008065C2    0x00000029  [ 11] literal string: http://sns.whalecloud.com/sina2/callback
</pre>

所以，若果你的加密 Key 以明文的形式写在文件里，是一件很危险的事情。

## dSYM 文件

我们在每次编译过后，都会生成一个 dsym 文件。dsym 文件中，存储了 16 进制的函数地址映射。

在 App 实际执行的二进制文件中，是通过地址来调用方法的。在 App crash 的时候，第三方工具（Fabric, 友盟等）会帮我们抓到崩溃的调用栈，调用栈里会包含 crash 地址的调用信息。然后，通过 dSYM 文件，我们就可以由地址映射到具体的函数位置。

XCode 中，选择 Window -> Organizer 可以看到我们生成的 archier 文件

![](https://mmbiz.qpic.cn/mmbiz_png/3QD99b9DjVHWud0atIQxpVMH4JtmOTHCrZFdEVZbNHFdEPEQTX4pdqlYeqPWSVNoldwicOMEfOKpMlwMsJtOREw/640?wx_fmt=png)

然后，

*   右键 -> 在 finder 中显示。

*   右键 -> 查看包内容。

关于如何用 dsym 文件来分析崩溃位置，可以查看我之前的一篇博客。

*   _iOS 如何调试第三方统计到的崩溃报告_ <sup>[1]</sup>

## 那些你想到和想不到的应用场景

### ****attribute****

或多或少，你都会在第三方库或者 iOS 的头文件中，见到过 **attribute**。 比如

<pre>__attribute__ ((warn_unused_result)) //如果没有使用返回值，编译的时候给出警告
</pre>

> `__attribtue__` 是一个高级的的编译器指令，它允许开发者指定更更多的编译检查和一些高级的编译期优化。

分为三种：

> 函数属性 （Function Attribute）
> 类型属性 (Variable Attribute)
> 变量属性 (Type Attribute)

语法结构

`__attribute__`语法格式为：**attribute**((attribute-list)) 放在声明分号 “;” 前面。

比如，在三方库中最常见的，声明一个属性或者方法在当前版本弃用了

<pre>@property (strong,nonatomic)CLASSNAME * property __deprecated;

</pre>

这样的好处是：给开发者一个过渡的版本，让开发者知道这个属性被弃用了，应当使用最新的 API，但是被__deprecated 的属性仍然可以正常使用。如果直接弃用，会导致开发者在更新 Pod 的时候，代码无法运行了。

**attribtue** 的使用场景很多，本文只列举 iOS 开发中常用的几个：

<pre>//弃用API，用作API更新
#define __deprecated    __attribute__((deprecated)) 

//带描述信息的弃用
#define __deprecated_msg(_msg) __attribute__((deprecated(_msg)))

//遇到__unavailable的变量/方法，编译器直接抛出Error
#define __unavailable    __attribute__((unavailable))

//告诉编译器，即使这个变量/方法 没被使用，也不要抛出警告
#define __unused    __attribute__((unused))

//和__unused相反
#define __used        __attribute__((used))

//如果不使用方法的返回值，进行警告
#define __result_use_check __attribute__((__warn_unused_result__))

//OC方法在Swift中不可用
#define __swift_unavailable(_msg)    __attribute__((__availability__(swift, unavailable, message=_msg)))
</pre>

## Clang 警告处理

你一定还见过如下代码：

<pre>#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wundeclared-selector"
///代码
#pragma clang diagnostic pop

</pre>

这段代码的作用是

1.  对当前编译环境进行压栈

2.  忽略 - Wundeclared-selector（未声明的）Selector 警告

3.  编译代码

4.  对编译环境进行出栈

通过 clang diagnostic push/pop, 你可以灵活的控制代码块的编译选项。

我在之前的一篇文章里，详细的介绍了 XCode 的警告相关内容。本文篇幅限制，就不详细讲解了。

*   _iOS 合理利用 Clang 警告来提高代码质量_ <sup>[2]</sup>

在这个链接，你可以找到所有的 Clang warnings 警告

*   fuckingclangwarnings

### 预处理

所谓预处理，就是在编译之前的处理。预处理能够让你定义编译器变量，实现条件编译。 比如，这样的代码很常见

<pre>#ifdef DEBUG
//...
#else
//...
#endif

</pre>

同样，我们同样也可以定义其他预处理变量, 在 XCode - 选中 Target-build settings 中，搜索 proprecess。然后点击图中蓝色的加号，可以分别为 debug 和 release 两种模式设置预处理宏。 比如我们加上：TestServer，表示在这个宏中的代码运行在测试服务器

![](https://mmbiz.qpic.cn/mmbiz_png/3QD99b9DjVHWud0atIQxpVMH4JtmOTHCQEVdaic2qsLibtbHqxQpRiaOSTzaIicKic6XMc2JZLktrgka1kicHukFZia5w/640?wx_fmt=png)

然后，配合多个 Target（右键 Target，选择 Duplicate），单独一个 Target 负责测试服务器。这样我们就不用每次切换测试服务器都要修改代码了。

<pre>#ifdef TESTMODE
//测试服务器相关的代码
#else
//生产服务器相关代码
#endif

</pre>

### 插入脚本

通常，如果你使用 CocoaPod 来管理三方库，那么你的 Build Phase 是这样子的：

![](https://mmbiz.qpic.cn/mmbiz_png/3QD99b9DjVHWud0atIQxpVMH4JtmOTHCfxfxNoP6vmS7OTsTAYRpXiczxI7lX9IfUQNHb52GwKKFWSJeNwuqllw/640?wx_fmt=png)

其中：[CP] 开头的，就是 CocoaPod 插入的脚本。

*   Check Pods Manifest.lock，用来检查 cocoapod 管理的三方库是否需要更新

*   Embed Pods Framework，运行脚本来链接三方库的静态 / 动态库

*   Copy Pods Resources，运行脚本来拷贝三方库的资源文件

而这些配置信息都存储在这个文件 (.xcodeprog) 里

![](https://mmbiz.qpic.cn/mmbiz_png/3QD99b9DjVHWud0atIQxpVMH4JtmOTHC9RsjmaiarYicgfw7VY5lQhO85pWE5mrrqv5wRczianEkiaYnfPpUYd1Rug/640?wx_fmt=png)

到这里，CocoaPod 的原理也就大致搞清楚了，通过修改 xcodeproject，然后配置编译期脚本，来保证三方库能够正确的编译连接。

同样，我们也可以插入自己的脚本，来做一些额外的事情。比如，每次进行 archive 的时候，我们都必须手动调整 target 的 build 版本，如果一不小心，就会忘记。这个过程，我们可以通过插入脚本自动化。

<pre>buildNumber=$(/usr/libexec/PlistBuddy -c "Print CFBundleVersion" "${PROJECT_DIR}/${INFOPLIST_FILE}")
buildNumber=$(($buildNumber + 1))
/usr/libexec/PlistBuddy -c "Set :CFBundleVersion $buildNumber" "${PROJECT_DIR}/${INFOPLIST_FILE}"

</pre>

这段脚本其实很简单，读取当前 pist 的 build 版本号, 然后对其加一，重新写入。

使用起来也很简单：

*   Xcode - 选中 Target - 选中 build phase

*   选择添加 Run Script Phase

![](https://mmbiz.qpic.cn/mmbiz_png/3QD99b9DjVHWud0atIQxpVMH4JtmOTHCguCmox9zbvOumd7SMib46eLfZsM0rGG6uibX7picBt9ibnl6ic251l5PaLg/640?wx_fmt=png)

*   然后把这段脚本拷贝进去，并且勾选 Run Script Only When installing，保证只有我们在安装到设备上的时候，才会执行这段脚本。重命名脚本的名字为 Auto Increase build number

*   然后，拖动这个脚本的到 Link Binary With Libraries 下面

![](https://mmbiz.qpic.cn/mmbiz_png/3QD99b9DjVHWud0atIQxpVMH4JtmOTHCOF1ysO1YUYOiaRg31eibEXkqfn74C2n3hzQRzeiawfKkIog2LrWo6aVbw/640?wx_fmt=png)

### 脚本编译打包

脚本化编译打包对于 CI（持续集成）来说，十分有用。iOS 开发中，编译打包必备的两个命令是：

<pre>//编译成.app
xcodebuild  -workspace $projectName.xcworkspace -scheme $projectName  -configuration $buildConfig clean build SYMROOT=$buildAppToDir
//打包
xcrun -sdk iphoneos PackageApplication -v $appDir/$projectName.app -o $appDir/$ipaName.ipa

通过info命令，可以查看到详细的文档
info xcodebuild

</pre>

_完整的脚本_ <sup>[3]</sup>，使用的时候，需要拷贝到工程的根目录

### 提高项目编译速度

通常，当项目很大，源代码和三方库引入很多的时候，我们会发现编译的速度很慢。在了解了 XCode 的编译过程后，我们可以从以下角度来优化编译速度：

**查看编译时间**

我们需要一个途径，能够看到编译的时间，这样才能有个对比，知道我们的优化究竟有没有效果。 对于 XCode 8，关闭 XCode，终端输入以下指令

<pre>$ defaults write com.apple.dt.Xcode ShowBuildOperationDuration YES
</pre>

然后，重启 XCode，然后编译，你会在这里看到编译时间。

![](https://mmbiz.qpic.cn/mmbiz_png/3QD99b9DjVHWud0atIQxpVMH4JtmOTHC2hsCj1JzauC9m7UenYmJErZKq65mURwicO0RqSqVotIHBcr0lTgZUqQ/640?wx_fmt=png)

代码层面的优化

**forward declaration**

所谓 forward declaration，就是 @class CLASSNAME，而不是 #import CLASSNAME.h。这样，编译器能大大提高 #import 的替换速度。

**对常用的工具类进行打包（Framework/.a）**

打包成 Framework 或者静态库，这样编译的时候这部分代码就不需要重新编译了。

**常用头文件放到预编译文件里**

XCode 的 pch 文件是预编译文件，这里的内容在执行 XCode build 之前就已经被预编译，并且引入到每一个. m 文件里了。

编译器选项优化

**Debug 模式下，不生成 dsym 文件**

上文提到了，dysm 文件里存储了调试信息，在 Debug 模式下，我们可以借助 XCode 和 LLDB 进行调试。所以，不需要生成额外的 dsym 文件来降低编译速度。

**Debug 开启 Build Active Architecture Only**

在 XCode -> Build Settings -> Build Active Architecture Only 改为 YES。这样做，可以只编译当前的版本，比如 arm7/arm64 等等，记得只开启 Debug 模式。这个选项在高版本的 XCode 中自动开启了。

**Debug 模式下，关闭编译器优化**

编译器优化

![](https://mmbiz.qpic.cn/mmbiz_png/3QD99b9DjVHWud0atIQxpVMH4JtmOTHCPmxjx5u62eyHFYDhTjsl1yvshy9JS5VWmOJ5vicunaUdBInmBmb8JLQ/640?wx_fmt=png)

### 参考

[1]http://blog.csdn.net/hello_hwc/article/details/50036323
[2]http://blog.csdn.net/Hello_Hwc/article/details/46425503
[3]https://github.com/LeoMobileDeveloper/Blogs/blob/master/DemoProjects/Scripts/autoIPA.sh

