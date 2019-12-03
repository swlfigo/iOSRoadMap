

# Mach-O 

Mach-O 格式全称为 Mach Object 文件格式的缩写，是 mac 上可执行文件的格式，类似于 windows 上的 PE 格式 (Portable Executable ), linux 上的 elf 格式 (Executable and Linking Format)。



![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-12-02-A4-2.png)

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-12-02-100546.png)

在 machO 这其中包含了很多的有效的信息，包括字符串，代码段，oc 类，oc 协议等各种的信息，利用这些信息我们也做到分析代码或者程序逻辑的作用

结合可知 Mach-O 文件包含了三部分内容：

- Header（头部），指明了 cpu 架构、大小端序、文件类型、Load Commands 个数等一些基本信息
- Load Commands（加载命令)，正如官方的图所示，描述了**怎样加载每个 Segment 的信息**。在 Mach-O 文件中可以有多个 Segment，每个 Segment 可能包含一个或多个 Section。
- Data（数据区），Segment 的具体数据，包含了代码和数据等。



## Headers

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-12-03-024432.jpg)

- magic 标志符 0xfeedface 是 32 位， 0xfeedfacf 是 64 位。
- cputype 和 cpusubtype 确定 cpu 类型、平台
- filetype 文件类型，可执行文件、符号文件（DSYM）、内核扩展等
- ncmds 加载 Load Commands 的数量
- flags dyld 加载的标志
  - `MH_NOUNDEFS` 目标文件没有未定义的符号，
  - `MH_DYLDLINK` 目标文件是动态链接输入文件，不能被再次静态链接,
  - `MH_SPLIT_SEGS` 只读 segments 和 可读写 segments 分离，
  - `MH_NO_HEAP_EXECUTION` 堆内存不可执行…



filetype 的定义有：

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-12-03-024558.jpg)

flags 的定义有：

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-12-03-030256.jpg)

**Headers** 能帮助校验 Mach-O 合法性和定位文件的运行环境。



## Load Commands

其占用的内存和加载命令的总数在 Headers 中已经指出

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-12-03-032034.jpg)

Load Commands 的定义:

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-12-03-032102.jpg)

- cmd 字段，如上图它指出了 command 类型
  - `LC_SEGMENT、LC_SEGMENT_64 ` 将 segment 映射到进程的内存空间，
  - `LC_UUID ` 二进制文件 id，与符号表 uuid 对应，可用作符号表匹配，
  - `LC_LOAD_DYLINKER` 启动动态加载器，
  - `LC_SYMTAB ` 描述在 `__LINKEDIT ` 段的哪找字符串表、符号表，
  - `LC_CODE_SIGNATURE` 代码签名等
- cmdsize 字段，主要用以计算出到下一个 command 的偏移量。