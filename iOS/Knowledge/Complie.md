# iOS 编译过程梳理



编译语言在执行的时候，必须先通过编译器生成机器码，机器码可以直接在 CPU 上执行，所以执行效率很高。

## 编译器的概述

编译器的作用是把我们的高级语言转换成机器可以识别的机器码，经典的设计结构如下：

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-04-12-072251.jpg)



- 前端（Frontend）：语法分析，语义分析和生成中间代码。在这个过程中，也会对代码进行检查，如果发现出错的或需要警告的会标注出来。
- 优化器（Optimizer）：会进行 BitCode 的生成，链接期优化等工作。
- 后端（Backend）：针对不同的架构，生成对应的机器码。

## Clang + LLVM 的编译过程

1. **预处理阶段**：import 头文件替换；macro 宏展开；处理预编译指令
2. **词法分析**：预处理完成后进入词法分析，将输入的代码转化为一系列符合特定语言的词法单元（token 流）。
3. **语法分析**：将词法分析得到的 token 流进行语法静态分析（Static Analysis），输出**抽象语法树（AST）**，过程中会校验语法是否错误。
4. **CodeGen 生成 IR 中间代码**：CodeGen 负责将语法树自顶向下遍历翻译成 `LLVM IR`，`IR` 是编译过程中前端的输出后端的输入。
5. **Optimize 优化 IR**：到这里 LLVM 会做一些优化工作，在 Xcode 的编译设置里可以设置优化级别 -01, -03, -0s，也可以写自己的 Pass，Pass 是 LLVM 优化工作的一个节点，一个节点做些事，一起加起来就构成了 LLVM 完整的优化和转化。附件：[官方 Pass 教程](http://llvm.org/docs/WritingAnLLVMPass.html)。
6. **LLVM Bitcode 生成字节码**：如果开启了 bitcode，苹果会做进一步优化。若有新的后端架构，依旧可以用这份优化过的 bitcode 去生成。
7. **生成汇编**
8. **生成目标文件**
9. **生成可执行文件**



## Xcode Build 的流程

我们在 Xcode 中使用 **Command + B** 或 **Command + R** 时，即完成了一次编译，来看下这个过程做了哪些事情。

编译过程分为四个步骤：

- 预编译（Pre-process）：宏替换、删除注释、展开头文件，产生 .i 文件。
- 编译（Compliling）：把前面生成的 .i 文件转化为汇编语言，产生 .s 文件。
- 汇编（Asembly）：把汇编语言 .s 文件转化为机器码文件，产生 .0 文件。
- 链接（Link）：对 .o 文件中的对于其他库的引用的地方进行引用，生成最后的可执行文件。也包括多个 .o 文件进行 link。

通过解析 Xcode 编译 log，可以发现 Xcode 是根据 Target 进行编译的。我们可以通过 Xcode 中的 Build Phases、Build Settings 及 Build Rules 来控制编译过程。

- Build Settings：这一栏下是对编译的细节进行设定，包含 build 过程的每个阶段的设置选项（包含编译、链接、代码签名、打包）。
- Build Phases：用于控制从源文件到可执行文件的整个过程，如编译哪些文件，编译过程中执行哪些自定义脚本。例如 CocoaPods 在这里会进行相关配置。
- Build Rules：指定了不同的文件类型该如何编译。一般我们不需要修改这里的内容。如果需要对特定类型的文件添加处理方法，可以在这里添加规则。

每个 Target 的具体编译过程也可以通过 log 日志获得。大致过程为：

- 编译信息写入辅助文件（如Entitlements.plist），创建编译后的文件架构
- 写入辅助信息（.hmap 文件）。将项目的文件结构对应表、将要执行的脚本、项目依赖库的文件结构对应表写成文件。
- 运行预设的脚本。如 Cocoapods 会在 Build Phases 中预设一些脚本（CheckPods Manifest.lock）。
- 编译 .m 文件，生成可执行文件 Mach-O。每次进行了 LLVM 的完整流程：前端（词法分析 - 语法分析 - 生成 IR）、优化器（优化 IR）、后端（生成汇编 - 生成目标文件 - 生成可执行文件）。使用 `CompileC` 和 `clang` 命令。
  CompileC 是 xcodebuild 内部函数的日志记录表示形式，它是 build.log 文件中有关编译的基本信息来源。
- 链接需要的库。如 Foundation.framework，AFNetworking.framework…
- 拷贝资源文件到目标包
- 编译 storyboard 文件
- 链接 storyboard 文件
- 编译 Asset 文件。如果使用 Asset.xcassets 来管理图片，这些图片会被编译为机器码，除了 icon 和 launchIamge。
- 处理 infoplist
- 执行 CocoaPods 脚本，将在编译项目前已编译好的依赖库和相关资源拷贝到包中。
- 拷贝 Swift 标准库
- 创建 .app 文件并对其签名



# Reference

[1 iOS 编译过程梳理](https://blog.jonyfang.com/2019/09/14/2019-09-14-ios-analyse-llvm/)



