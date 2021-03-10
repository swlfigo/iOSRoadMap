

# MachO

## MachO 文件

`Mach-O` 其实是 `Mach Object` 文件格式的缩写，是 `mac` 以及 `iOS` 上可执行文件的格式， 类似于 `windows` 上的 `PE` 格式 ( Portable Executable ) , `linux` 上的 `elf` 格式 ( Executable and Linking Format )  .

它是一种用于可执行文件、目标代码、动态库的文件格式。作为 `a.out` 格式的替代，`Mach-O` 提供了更强的扩展性。

但是除了可执行文件外 , 其实还有一些文件也是使用的 `Mach-O` 的文件格式 .

属于 `Mach-O` 格式的常见文件

> - 目标文件 .o
> - 库文件
>   - .a
>   - .dylib
>   - Framework
> - 可执行文件
> - dyld ( 动态链接器 )
> - .dsym ( 符号表 )



使用 `file` 命令可以查看文件类型

也就是说 `Mach-O` 并非一定是可执行文件 , 它是一种文件格式 , 分为 `Mach-O Object` 目标文件 、 `Mach-O executable 可执行文件、 `Mach-O dynamically` 动态库文件、 `Mach-O dynamic linker` 动态链接器文件、 `Mach-O dSYM companion` 符号表文件 , 等等 .

还看到一个 `arm64` , 这个是什么意思呢 ?

> - 在 release 模式下
> - 支持 iOS 11.0 系统版本以下

**当满足这两个条件时 , 我们的应用打包出来的 `Mach-O ececutable` 可执行文件是包含 `arm64` 以及 `arm_v7` 的架构的** , `iPhone 5C` 以上机型都是 `64` 位系统了 .

那么包含了支持多架构的 `Mach-O executable` 可执行文件被称为 : **通用二进制文件** , 即多种架构都可读取运行 .

另外 `Xcode` 中通过编译设置 `Architectures` 是可以更改所生成的 `Mach-O executable` 可执行文件的支持架构的 .

![img](https://tva1.sinaimg.cn/large/008eGmZEgy1goen5ar14aj30zk0a1wfv.jpg)

> 编译器在生成 `Mach-O` 文件会选择 `Architectures` 以及 `Valid Architectures` 的交集 , 因此想要支持多架构的话 , 在`Valid Architectures` 中继续添加就可以了 , 编译生成 `Mach-O` 之后 , 使用 `file` 命令可以检查下结果 .

### 


作者：李斌同学
链接：https://juejin.cn/post/6844903983841214472
来源：掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。作者：李斌同学
链接：https://juejin.cn/post/6844903983841214472
来源：掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。