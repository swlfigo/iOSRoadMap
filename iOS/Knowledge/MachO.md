

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



### Load Commands

`Load Commands` 详细保存着加载指令的内容 , 告诉链接器如何去加载这个 `Mach-O` 文件.

通过查看内存地址我们发现 , 在内存中 , `Load Commands` 是紧跟在 `Mach_header` 之后的 .

那么这些 `Load Commands` 对应了什么呢 ? 我们以 arm64 为例.

![img](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-03-11-092013.jpg)

其中 **_TEXT** 段和 **_DATA** 段 , 是我们经常需要研究的 , `MachOView` 下面也有详细列出.

![img](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-03-11-092018.jpg)

### 

### _TEXT 段

我们来看看 `_TEXT` 段里都存放了什么 , 其实真正开始读取就是从 `_TEXT` 段开始读取的 .

| 名称                      | 内容             |
| ------------------------- | ---------------- |
| `_text`                   | 主程序代码       |
| `_stubs` , `_stub_helper` | 动态链接         |
| `_objc_methodname`        | 方法名称         |
| `_objc_classname`         | 类名称           |
| `_objc_methtype`          | 方法类型 ( v@: ) |
| `_cstring`                | 静态字符串常量   |

### _DATA 段

`_DATA` 在内存中是紧跟在 `_TEXT` 段之后的.

| 名称                                    | 内容           |
| --------------------------------------- | -------------- |
| `_got` : Non-Lazy Symbol Pointers       | 非懒加载符号表 |
| `_la_symbol_ptr` : Lazy Symbol Pointers | 懒加载符号表   |
| `_objc_classlist`                       | 类列表         |



下面列举一些常见的 Section。

| Section                   | 用途                                                         |
| ------------------------- | ------------------------------------------------------------ |
| `__TEXT.__text`           | 主程序代码                                                   |
| `__TEXT.__cstring`        | C 语言字符串                                                 |
| `__TEXT.__const`          | `const` 关键字修饰的常量                                     |
| `__TEXT.__stubs`          | 用于 Stub 的占位代码，很多地方称之为*桩代码*。               |
| `__TEXT.__stubs_helper`   | 当 Stub 无法找到真正的符号地址后的最终指向                   |
| `__TEXT.__objc_methname`  | Objective-C 方法名称                                         |
| `__TEXT.__objc_methtype`  | Objective-C 方法类型                                         |
| `__TEXT.__objc_classname` | Objective-C 类名称                                           |
| `__DATA.__data`           | 初始化过的可变数据                                           |
| `__DATA.__la_symbol_ptr`  | lazy binding 的指针表，表中的指针一开始都指向 `__stub_helper` |
| `__DATA.nl_symbol_ptr`    | 非 lazy binding 的指针表，每个表项中的指针都指向一个在装载过程中，被动态链机器搜索完成的符号 |
| `__DATA.__const`          | 没有初始化过的常量                                           |
| `__DATA.__cfstring`       | 程序中使用的 Core Foundation 字符串（`CFStringRefs`）        |
| `__DATA.__bss`            | BSS，存放为初始化的全局变量，即常说的静态内存分配            |
| `__DATA.__common`         | 没有初始化过的符号声明                                       |
| `__DATA.__objc_classlist` | Objective-C 类列表                                           |
| `__DATA.__objc_protolist` | Objective-C 原型                                             |
| `__DATA.__objc_imginfo`   | Objective-C 镜像信息                                         |
| `__DATA.__objc_selfrefs`  | Objective-C `self` 引用                                      |
| `__DATA.__objc_protorefs` | Objective-C 原型引用                                         |
| `__DATA.__objc_superrefs` | Objective-C 超类引用                                         |



## Reference

[1. Macho-O文件](https://juejin.cn/post/6844903983841214472)

[2.Mach-O 文件格式探索](https://www.desgard.com/iOS-Source-Probe/C/mach-o/Mach-O%20%E6%96%87%E4%BB%B6%E6%A0%BC%E5%BC%8F%E6%8E%A2%E7%B4%A2.html)

