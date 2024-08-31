# 引用计数器

## 引用计数的存储策略

1. 有些对象如果支持使用`Tagged Pointer`，苹果会直接将其指针值作为引用计数返回；
2. 如果当前设备是`64`位环境并且使用`Objective-C 2.0`，那么“一些”对象会使用其`isa`指针的一部分空间来存储它的引用计数；
3. 否则`Runtime`会使用一张散列表来管理引用计数。



## Tagged Pointer



[Tagged Pointer](./TaggedPointer.md)



## isa指针



[isa](../Objc_Object/isa.md)



为什么既要使用一个`extra_rc`又要使用`SideTables`？

可能是因为历史问题，以前cpu是`32`位的，`isa`中能存储的引用计数就只有$2^7=128$。因此在`arm64`下，引用计数通常是存储在`isa`中的。



## SideTable



[SideTable](./SideTables.md)



### alloc实现

经过一系列调用，最终调用了C函数calloc,此时**并没有设置引用计数为1**

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





## 引用计数的获取

通过`retainCount`可以获取到引用计数器，其定义：

```
- (NSUInteger)retainCount {
    return ((id)self)->rootRetainCount();
}

inline uintptr_t objc_object::rootRetainCount() {
    if (isTaggedPointer()) return (uintptr_t)this;

    sidetable_lock();
    // 加锁，用汇编指令ldxr来保证原子性
    isa_t bits = LoadExclusive(&isa.bits);
    // 释放锁，使用汇编指令clrex
    ClearExclusive(&isa.bits);
    if (bits.nonpointer) {
        uintptr_t rc = 1 + bits.extra_rc;
        if (bits.has_sidetable_rc) {
            rc += sidetable_getExtraRC_nolock();
        }
        sidetable_unlock();
        return rc;
    }

    sidetable_unlock();
    return sidetable_retainCount();
}

//sidetable_retainCount()函数实现
uintptr_t objc_object::sidetable_retainCount() {
    SideTable& table = SideTables()[this];

    size_t refcnt_result = 1;
    
    table.lock();
    RefcountMap::iterator it = table.refcnts.find(this);
    if (it != table.refcnts.end()) {
        // this is valid for SIDE_TABLE_RC_PINNED too
        refcnt_result += it->second >> SIDE_TABLE_RC_SHIFT;
    }
    table.unlock();
    return refcnt_result;
}
```

从上面的代码可知，获取引用计数的时候分为三种情况：

1. `Tagged Pointer`的话，直接返回isa本身；
2. 非`Tagged Pointer`，且开启了指针优化，此时引用计数先从`extra_rc`中去取（这里将取出来的值进行了+1操作，所以在存的时候需要进行-1操作），接着判断是否有`SideTable`，如果有再加上存在`SideTable`中的计数；
3. 非`Tagged Pointer`，没有开启了指针优化，使用`sidetable_retainCount()`函数返回。



## 总结

1. 引用计数存在什么地方？

   - `Tagged Pointer`不需要引用计数，苹果会直接将对象的指针值作为引用计数返回；
   - 开启了指针优化（`nonpointer == 1`）的对象其引用计数优先存在`isa`的`extra_rc`中，大于`524288`便存在`SideTable`的`RefcountMap`或者说是`DenseMap`中；
   - 没有开启指针优化的对象直接存在`SideTable`的`RefcountMap`或者说是`DenseMap`中。

2. retain/release的实质

   - `Tagged Pointer`不参与`retain`/`release`；
   - 找到引用计数存储区域，然后+1/-1，并根据是否开启指针优化，处理进位/借位的情况；
   - 当引用计数减为0时，调用`dealloc`函数。

3. isa是什么

   ```
   // ISA() assumes this is NOT a tagged pointer object
   Class ISA();
   
   // getIsa() allows this to be a tagged pointer object
   Class getIsa();
   ```

   - 首先要知道，isa指针已经不一定是类指针了，所以需要用`ISA()`获取类指针；
   - `Tagged Pointer`的对象没有`isa`指针，有的是`isa_t`的结构体；
   - 其他对象的isa指针还是类指针。

4. 对象的值是什么

   - 如果是`Tagged Pointer`，对象的值就是指针；
   - 如果非`Tagged Pointer`， 对象的值是指针指向的内存区域中的值。





## 补充: 一道多线程安全的题目

以下代码运行结果

```
@property (nonatomic, strong) NSString *target;
//....

dispatch_queue_t queue = dispatch_queue_create("parallel", DISPATCH_QUEUE_CONCURRENT);
for (int i = 0; i < 1000000 ; i++) {
    dispatch_async(queue, ^{
        self.target = [NSString stringWithFormat:@"ksddkjalkjd%d",i];
    });
}
```

答案：大概率地发生Crash。

Crash的原因：过度释放。

这道题看着虽然是多线程范围的，但是解题的最重要思路确是在引用计数上，更准确的来说是看对强引用的理解程度。关键知识点如下：

1. 全局队列和自定义并行队列在异步执行的时候会根据任务系统决定开辟线程个数；
2. `target`使用`strong`进行了修饰，Block是会截获对象的修饰符的；
3. 即使使用`_target`效果也是一样，因为默认使用`strong`修饰符隐式修饰；
4. `strong`的源代码如下：

```
objc_storeStrong(id *location, id obj) {
    id prev = *location;
    if (obj == prev) {
        return;
    }
    objc_retain(obj);
    *location = obj;
    objc_release(prev);
}
```

假设这个并发队列创建了两个线程A和B，由于是异步的，可以同时执行。因此会出现这么一个场景，在线程A中，代码执行到了`objc_retain(obj)`，但是在线程B中可能执行到了`objc_release(prev)`，此时`prev`已经被释放了。那么当A在执行到`objc_release(prev)`就会过度释放，从而导致程序crash。

解决方法：

1. 加个互斥锁
2. 使用串行队列，使用串行队列的话，其实内部是靠`DISPATCH_OBJ_BARRIER_BIT`设置阻塞标志位
3. 使用weak
4. Tagged Pointer，如果说上面的`self.target`指向的是一个`Tagged Pointer`技术的`NSString`，那程序就没有问题。





## Reference

[1.iOS引用计数管理之揭秘计数存储](https://juejin.im/entry/6844903636183547918)

[OC内存管理-引用计数器 | NeroXie的个人博客](https://www.neroxie.com/2018/12/02/OC内存管理-引用计数器/)
