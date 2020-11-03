# SideTables 散列表

**NONPOINTER_ISA** 这个设计思想跟TaggetPointer类似，ISA其实并不单单是一个指针。其中一些位仍旧编码指向对象的类。但是实际上并不会使用所有的地址空间，Objective-C 运行时会使用这些额外的位去存储每个对象数据就像它的引用计数和是否它已经被弱引用。

查看isa的定义,它里面定义了一个位域：ISA_BITFIELD，点击查看这个宏：

```objective-c
# if __arm64__
#   define ISA_BITFIELD                                                      \
      uintptr_t nonpointer        : 1;                                       \
      uintptr_t has_assoc         : 1;                                       \
      uintptr_t has_cxx_dtor      : 1;                                       \
      uintptr_t shiftcls          : 33; /*MACH_VM_MAX_ADDRESS 0x1000000000*/ \
      uintptr_t magic             : 6;                                       \
      uintptr_t weakly_referenced : 1;                                       \
      uintptr_t deallocating      : 1;                                       \
      uintptr_t has_sidetable_rc  : 1;                                       \
      uintptr_t extra_rc          : 19
# elif __x86_64__
#   define ISA_BITFIELD                                                        \
      uintptr_t nonpointer        : 1;                                         \
      uintptr_t has_assoc         : 1;                                         \
      uintptr_t has_cxx_dtor      : 1;                                         \
      uintptr_t shiftcls          : 44; /*MACH_VM_MAX_ADDRESS 0x7fffffe00000*/ \
      uintptr_t magic             : 6;                                         \
      uintptr_t weakly_referenced : 1;                                         \
      uintptr_t deallocating      : 1;                                         \
      uintptr_t has_sidetable_rc  : 1;                                         \
      uintptr_t extra_rc          : 8

```

可以看到，__x86_64__和__arm64__下的位域定义是不一样的，不过都是占满了所有的64位（1+1+1+33+6+1+1+1+19 = 64，__x86_64__同理），

```objective-c
union isa_t {  
  Class cls;  ...  (还有很多其他的成员，包括引用计数数量) 
}
```

 **nonpointer**：表示是否对isa开启指针优化 。0代表是纯isa指针，1代表除了地址外，还包含了类的一些信息、对象的引用计数等。 如果该实例对象启用了Non-pointer，那么会对isa的其他成员赋值，否则只会对cls赋值。



是否关闭Non-pointer目前有这么几个判断条件，这些都可以在runtime源码objc-runtime-new.m中找到逻辑。

```objective-c
1：包含swift代码；
2：sdk版本低于10.11；
3：runtime读取image时发现这个image包含__objc_rawisa段；
4：开发者自己添加了OBJC_DISABLE_NONPOINTER_ISA=YES到环境变量中；
5：某些不能使用Non-pointer的类，GCD等；
6：父类关闭。
```



**has_assoc**：关联对象标志位 

**has_cxx_dtor**：该对象是否有C++或Objc的析构器，如果有析构函数，则需要做一些析构的逻辑处理，如果没有，则可以更快的释放对象 

**shiftcls**：存在类指针的值，开启指针优化的情况下，arm64位中有33位来存储类的指针

**magic**：判断当前对象是真的对象还是一段没有初始化的空间

**weakly_referenced**：是否被指向或者曾经指向一个ARC的弱变量，没有弱引用的对象释放的更快

**deallocating**：是否正在释放 

**has_sidetable_rc**：当对象引用计数大于10时，则需要进位

 **extra_rc**：表示该对象的引用计数值，实际上是引用计数减一。例如：如果引用计数为10，那么extra_rc为9。~~如果引用计数大于10，则需要使用**has_sidetable_rc**~~

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2020-11-02-094715.png)

## extra_rc

那引用计数存在哪里呢？秘密就在`extra_rc`中。

> extra_rc只是存储了额外的引用计数，实际的引用计数计算公式：`引用计数=extra_rc+1`。

`extra_rc`占了19位，可以存储的最大引用计数：$2^{19}-1+1=524288$，超过它就需要进位到`SideTables`。SideTables是一个Hash表，根据对象地址可以找到对应的`SideTable`，`SideTable`内包含一个`RefcountMap`，根据对象地址取出其引用计数，类型是`size_t`。
它是一个`unsigned long`，最低两位是标志位，剩下的62位用来存储引用计数。我们可以计算出引用计数的理论最大值：$2^{62+19}=2.417851639229258e24$。

> 其实isa能存储的524288在日常开发已经完全够用了，为什么还要搞个Side Table？我猜测是因为历史问题，以前cpu是32位的，isa中能存储的引用计数就只有$2^{7}=128$。因此在arm64下，引用计数通常**是存储在isa中**的。



对象的引用计数到底存哪里了

```shell
1：对象是否是Tagged Pointer对象；
2：对象是否启用了Non-pointer；
3：对象未启用Non-pointer。
```

满足1则不判断2，依次类推。



### 散列表  SideTables

在runtime内存空间中，SideTables是一个hash数组，里面存储了SideTable。SideTables的hash键值就是一个对象obj的address。 因此可以说，一个obj，对应了一个SideTable。但是一个SideTable，会对应多个obj。因为SideTable的数量有限，所以会有很多obj共用同一个SideTable。



**如果该对象不是Tagged Pointer且关闭了Non-pointer，那该对象的引用计数就使用SideTable来存。**



`SideTables`是一个~~64个元素长度~~8个元素长度 的hash数组，里面存储了`SideTable`。`SideTables`的hash键值就是一个对象`obj`的`address`。

value包含了 引用计数与弱引用表




![image-20190324165113450](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-24-085318.jpg)



SideTable的结构

```objective-c
struct SideTable {
    spinlock_t slock;      // 自旋锁
    RefcountMap refcnts;    //引用计数的Map表 key-value
    weak_table_t weak_table;  //弱引用表
```

![image-20190324165155926](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-24-085324.jpg)



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

执行**table = &SideTables()[obj];\**之后，执行到了\**array[indexForPointer(p)].value;**，然后进行哈希算法获取到下标，再返回所需的sideTable

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2020-11-02-093727.jpg)





### **为什么不直接用一张SideTable，而是用SideTables去管理多个SideTable？** 

SideTable里有一个自旋锁，如果把所有的类都放在同一个SideTable，有任何一个类有改动都会对整个table做操作，并且在操作一个类的同时，操作别的类会被锁住等待，这样会导致操作效率和查询效率都很低。而有多个SideTable的话，操作的都是单个Table，并不会影响其他的table，这就是分离锁。





## Reference



[1. NONPOINTER_ISA和散列表 ](https://juejin.im/post/6844903885522337805)

[2. Exploring the nature of Objective-C reference counting](https://programmer.help/blogs/exploring-the-nature-of-objective-c-reference-counting.html)

[3. 探寻Objective-C引用计数本质](https://crmo.github.io/2018/05/26/%E6%8E%A2%E5%AF%BBObjective-C%E5%BC%95%E7%94%A8%E8%AE%A1%E6%95%B0%E6%9C%AC%E8%B4%A8/)



