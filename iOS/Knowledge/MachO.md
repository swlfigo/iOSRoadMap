

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

也就是说 `Mach-O` 并非一定是可执行文件 , 它是一种文件格式 , 分为 `Mach-O Object` 目标文件 、 `Mach-O ececutable` 可执行文件、 `Mach-O dynamically` 动态库文件、 `Mach-O dynamic linker` 动态链接器文件、 `Mach-O dSYM companion` 符号表文件 , 等等 .

