# SideTables 散列表



# SideTables



## 简介

SideTables 是一个 **哈希数组** 包含 64 个元素，里面存储了`SideTable`,元素的内容为 `SideTable` 的地址，每一个 SideTable 又包含有一个自选锁、一张全局的引用计数表、一张全局的弱引用表。



在runtime内存空间中，SideTables是一个hash数组，里面存储了SideTable。SideTables的hash键值就是一个对象obj的address。 因此可以说，一个obj，对应了一个SideTable。但是一个SideTable，会对应多个obj。因为SideTable的数量有限，所以会有很多obj共用同一个SideTable。



## 结构



**如果该对象不是Tagged Pointer且关闭了Non-pointer，那该对象的引用计数就使用SideTable来存。**






![image-20190324165113450](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-24-085318.jpg)



SideTable的结构

```objective-c
struct SideTable {
    spinlock_t slock;      // 自旋锁
    RefcountMap refcnts;    //引用计数的Map表 key-value
    weak_table_t weak_table;  //弱引用表
```

![image-20190324165155926](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-24-085324.jpg)





### SideTable 的锁 slock

```
spinlock_t slock;
```

使用的是 **自旋锁**，而且是 **非公平 `unfair`** 锁。

自旋锁 - 忙等待，直到锁被释放（区别于互斥锁的休眠等待）。

非公平锁 - 获取锁的顺序和申请的顺序无关，即可能 A 线程第一个申请锁，却在 B、C 获得锁之后 A 才获得锁。





### SideTable 的引用计数表 refcnts

```
RefcountMap refcnts;
```

哈希表，key 为 `objc_object`，即 OC 对象，value 为引用计数。

当 value 为 0 的时候，会将该记录从表中移除。



### SideTable 的弱引用表 weak_table

```
weak_table_t weak_table;
```

`weak_table_t` 是一个哈希结构体，其结构如下：

```
1/**
2 * The global weak references table. Stores object ids as keys,
3 * and weak_entry_t structs as their values.
4 */
5struct weak_table_t {
6    weak_entry_t *weak_entries;
7    size_t    num_entries;
8    uintptr_t mask;
9    uintptr_t max_hash_displacement;
10};
```

其中第一个成员 `weak_entries` 存放着若干个数据，其余的成员都是用来做哈希定位的，

哈希数据使用 `weak_entry_t` 结构体保存，定义如下：

```
1struct weak_entry_t {
2    DisguisedPtr<objc_object> referent;
3    union {
4        struct {
5            weak_referrer_t *referrers;
6            uintptr_t        out_of_line_ness : 2;
7            uintptr_t        num_refs : PTR_MINUS_2;
8            uintptr_t        mask;
9            uintptr_t        max_hash_displacement;
10        };
11        struct {
12            // out_of_line_ness field is low bits of inline_referrers[1]
13            weak_referrer_t  inline_referrers[WEAK_INLINE_COUNT];
14        };
15    };
16    ......
17}
```

- `referent` 被引用对象的 **内存地址**
- `referrers` & `inline_referrers` （所有指向被引用对象的）弱引用指针

可以看出来 `weak_entry_t` 可以理解成一个字典结构，将 **被引用对象的内存地址作为 key**，**所有指向它的弱引用指针数组作为 value**，保存着 **某个对象所有指向它的 weak 指针**。

在对 `weak_table_t` 进行哈希查找的时候，会将要查找的对象地址作为参数，通过 mask，去对比表中每个 `weak_entry_t` 的 `referent`，找到对应的 `weak_entry_t`，然后对其弱引用指针进行操作。





### 获取SideTable



如何从sideTables里找到特定的sideTable呢，这就用到了散列函数。runtime是通过这么一个函数来获取到相应的sideTable：



```objective-c

table = &SideTables()[obj];


static StripedMap<SideTable>& SideTables() {
    return *reinterpret_cast<StripedMap<SideTable>*>(SideTableBuf);
}

```

```objective-c
template<typename T>
class StripedMap {
#if TARGET_OS_IPHONE && !TARGET_OS_SIMULATOR
    enum { StripeCount = 8 }; // iPhone时这个值为8
#else
    enum { StripeCount = 64 }; //否则为64
#endif

    struct PaddedT {
        T value alignas(CacheLineSize);
    };

    PaddedT array[StripeCount];

    static unsigned int indexForPointer(const void *p) {
        //这里是做类型转换
        uintptr_t addr = reinterpret_cast<uintptr_t>(p);

        //这就是哈希算法了
        return ((addr >> 4) ^ (addr >> 9)) % StripeCount;
    }
public:
    T& operator[] (const void *p) { 
        //返回sideTable
        return array[indexForPointer(p)].value; 
    }

```

可以看到，在对StripeCount取余后，所得到的值根据机器不同，会在0-7或者0-63之间，这就是通过哈希函数来获取到了sideTable的下标，然后再根据value取到所需的sideTable。

执行table = &SideTables()[obj];之后，执行到了array[indexForPointer(p)].value;，然后进行哈希算法获取到下标，再返回所需的sideTable

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2020-11-02-093727.jpg)





### **为什么不直接用一张SideTable，而是用SideTables去管理多个SideTable？** 

SideTable里有一个自旋锁，如果把所有的类都放在同一个SideTable，有任何一个类有改动都会对整个table做操作，并且在操作一个类的同时，操作别的类会被锁住等待，这样会导致操作效率和查询效率都很低。而有多个SideTable的话，操作的都是单个Table，并不会影响其他的table，这就是分离锁。





## Reference



[1. NONPOINTER_ISA和散列表 ](https://juejin.im/post/6844903885522337805)

[2. Exploring the nature of Objective-C reference counting](https://programmer.help/blogs/exploring-the-nature-of-objective-c-reference-counting.html)

[3. 探寻Objective-C引用计数本质](https://crmo.github.io/2018/05/26/%E6%8E%A2%E5%AF%BBObjective-C%E5%BC%95%E7%94%A8%E8%AE%A1%E6%95%B0%E6%9C%AC%E8%B4%A8/)



