

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



### 通用二进制文件

- 苹果公司提出的一种程序代码。能同时适用多种架构的二进制文件
- 同一个程序包中同时为多种架构提供最理想的性能。
- 因为需要储存多种代码，通用二进制应用程序通常比单一平台二进制的程序要大。
- 但是由于两种架构有共通的非执行资源，所以并不会达到单一版本的两倍之多。
- 而且由于执行中只调用一部分代码，运行起来也不需要额外的内存。

通用二进制文件通常被称为 `Universal binary` , 在 `MachOView` 等 中叫做 `Fat binary` , **这种二进制文件是可以完全拆分开来 , 或者重新组合的**



## Mach-O 文件结构



![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-12-02-A4-2.png)

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-12-02-100546.png)

`Mach-O` 的组成结构如图所示包括了

- `Header` 包含该二进制文件的一般信息
  - 字节顺序、架构类型、加载指令的数量等。
  - 使得可以快速确认一些信息，比如当前文件用于 `32` 位还是 `64` 位，对应的处理器是什么、文件类型是什么
- `Load commands` 一张包含很多内容的表
  - 内容包括区域的位置、符号表、动态符号表等。
- `Data` 通常是对象文件中最大的部分
  - 包含 `Segement` 的具体数据

### Mach Header

![img](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-03-10-151441.jpg)

`Header` 中存储的内容大致如上图所示 , 那么每一条到底对应着什么呢 ? , 我们打开源码看一下, `cmd + shift + o` , 搜索 `load.h` , 找 `mach_header_64` 结构体.

```c++
struct mach_header_64 {
    uint32_t	magic;		/* 魔数,快速定位64位/32位 */
    cpu_type_t	cputype;	/* cpu 类型 比如 ARM */
    cpu_subtype_t	cpusubtype;	/* cpu 具体类型 比如arm64 , armv7 */
    uint32_t	filetype;	/* 文件类型 例如可执行文件 .. */
    uint32_t	ncmds;		/* load commands 加载命令条数 */
    uint32_t	sizeofcmds;	/* load commands 加载命令大小*/
    uint32_t	flags;		/* 标志位标识二进制文件支持的功能 , 主要是和系统加载、链接有关*/
    uint32_t	reserved;	/* reserved , 保留字段 */
};
```

`mach_header_64` 相较于 `mach_header` , 也就是 `32` 位头文件 , 只是多了一个保留字段 . `mach_header` 是链接器加载时最先读取的内容 , 它决定了一些基础架构 , 系统类型 , 指令条数等信息.


作者：李斌同学
链接：https://juejin.cn/post/6844903983841214472
来源：掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。