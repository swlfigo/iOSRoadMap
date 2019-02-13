# 20 Memory内存管理

## 内存管理方案
* 系统针对不同场景下内存管理方案不同，如小对象(NSNumber等)采用 `TaggedPointer` 内存管理方案.
* 针对64位下iOS程序采用 `NONPOINTER_ISA` 方案
* 散列表(包括引用计数表，弱引用表)

## NONPOINTER_ISA
* 第一位的 0 或 1 代表是纯地址型 isa 指针，还是 NONPOINTER_ISA 指针。
* 第二位，代表是否有关联对象
* 第三位代表是否有 C++ 代码。
* 接下来33位代表指向的内存地址
* 接下来有 弱引用 的标记
* 接下来有是否 delloc 的标记....等等

## 什么是ARC

* ARC 是LLVM和Runtime共同协作的结果，进行自动引用计数的管理
* ARC中禁止手动调用retain/release/retaincount/dealloc
* ARC中新增weak，strong属性关键字

## 引用计数管理
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
![deallo](http://img.isylar.com/media/dealloc.png)

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

![](http://img.isylar.com/media/15475622619941.jpg)

