# Memory内存管理

## 1.内存中的5大区分别是什么？
* 栈区（stack）：由编译器自动分配释放 ，存放函数的参数值，局部变量的值等。其 操作方式类似于数据结构中的栈。
* 堆区（heap）：一般由程序员分配释放， 若程序员不释放，程序结束时可能由OS回收 。注意它与数据结构中的堆是两回事，分配方式倒是类似于链表。
* 全局区（静态区）（static）：全局变量和静态变量的存储是放在一块的，初始化的 全局变量和静态变量在一块区域， 未初始化的全局变量和未初始化的静态变量在相邻的另一块区域。 - 程序结束后由系统释放。
* 文字常量区：常量字符串就是放在这里的。 程序结束后由系统释放。
* 程序代码区：存放函数体的二进制代码。

![image-20190324161722448](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-24-085314.jpg)


## 2.内存管理方案
* 系统针对不同场景下内存管理方案不同，如小对象(NSNumber等)采用 `TaggedPointer` 内存管理方案.
* 针对64位下iOS程序采用 `NONPOINTER_ISA` 方案
* 散列表(包括引用计数表，弱引用表)

#### 2.1 NONPOINTER_ISA
* 第一位的 0 或 1 代表是纯地址型 isa 指针，还是 NONPOINTER_ISA 指针。
* 第二位，代表是否有关联对象
* 第三位代表是否有 C++ 代码。
* 接下来33位代表指向的内存地址
* 接下来有 弱引用 的标记
* 接下来有是否 delloc 的标记....等等

#### 2.2 散列表

![image-20190324165113450](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-24-085318.jpg)

![image-20190324165155926](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-24-085324.jpg)



SideTable是一张哈希表.使用多张 SideTable .因为可以提高查找效率.对象的引用计数值修改时候可能存在不同线程，需要锁保证数据安全，如果用同一张表会导致效率问题，需要使用`分离锁`分离操作.



## 3.数据结构



![image-20190324165929441](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-24-085931.png)

## 4. 什么是ARC

* ARC 是LLVM和Runtime共同协作的结果，进行自动引用计数的管理
* ARC中禁止手动调用retain/release/retaincount/dealloc
* ARC中新增weak，strong属性关键字

## 5.引用计数管理
### alloc实现
经过一系列调用，最终调用了C函数calloc,此时并没有设置引用计数为1

### retain实现

```
SideTable &table = SideTables()[this];
//在tables里面，根据当前对象指针获取对应的sidetable

size_t &refcntStorage = table.refcnts[this];
//获得引用计数

//添加引用计数
refcntStorage += SIDE_TABLE_RC_ONE(4,位计算)
```

### release 实现

```
SideTable &table = SideTables()[this];
RefcountMap::iterator it = table.refcnts.find[this];
it->second -= SIDE_TABLE_RC_ONE
```

### retianCount

```
SideTable &table = SideTables()[this];
size_t refcnt_result = 1;
RefcountMap::iterator it = table.refcnts.find[this];
refcnt_result += it->second >> SIDE_TABLE_RC_SHIFT;(将向右偏移操作)
```

### dealloc实现
![deallo](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-07-23-134647.png)

## 弱引用管理

```
{
    id __weak obj1 = ob1; 
    //  => 编译 
    id objc1;
    objc_initWeak(&obj1,obj);
    //obj1弱引用对象地址，obj被修饰对象
}
```

添加weak变量

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-07-23-134710.jpg)



![image-20190324170107697](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-24-090109.png)

