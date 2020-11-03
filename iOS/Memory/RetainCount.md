# RetainCount

引用计数在 [SideTables](./SideTables.md) 中有介绍

## 引用计数管理

### alloc实现

经过一系列调用，最终调用了C函数calloc,此时并**没有设置引用计数为1**

### **Tagged Pointer对象**

retain时。

```
id objc_object::rootRetain(bool tryRetain, bool handleOverflow) {
    if (isTaggedPointer()) return (id)this;
    ...
}
```

release时。

```
bool  objc_object::rootRelease(bool performDealloc, bool handleUnderflow) {
    if (isTaggedPointer()) return false;
    ...
}
```

retainCount时。

```
uintptr_t objc_object::rootRetainCount() {
    if (isTaggedPointer()) return (uintptr_t)this;
    ...
}
```

由此可见对于Tagged Pointer对象，并没有任何的引用计数操作，引用计数数量也只是单纯的返回自己地址罢了。



### **开启了Non-pointer**

retain时。

```
id objc_object::rootRetain(bool tryRetain, bool handleOverflow) {
    ...
    //其实就是对isa的extra_rc变量进行+1，前面说到isa会存很多东西
    addc(newisa.bits, 1, 0, &carry);
    ...
}
```

release时。

```
bool  objc_object::rootRelease(bool performDealloc, bool handleUnderflow) {
    ...
    //其实就是对isa的extra_rc变量进行-1
    subc(newisa.bits, 1, 0, &carry);
    ...
}
```

retainCount时。

```
uintptr_t objc_object::rootRetainCount() {
    ...
    //其实就是获取isa的extra_rc值再+1，alloc新建一个对象时bits.extra_rc为0并不是1，这个要注意。
    uintptr_t rc = 1 + bits.extra_rc;
    ...
}
```



### **未开启Non-pointer isa**

### alloc实现

经过一系列调用，最终调用了C函数calloc,此时**并没有设置引用计数为1**

此时**并没有设置引用计数为1**

此时**并没有设置引用计数为1**

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

### 

## Reference

[1.iOS引用计数管理之揭秘计数存储](https://juejin.im/entry/6844903636183547918)