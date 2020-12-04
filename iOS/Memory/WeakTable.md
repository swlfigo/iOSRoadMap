# weak的实现原理

> 使用场景都比较清晰，避免出现对象之间的强强引用而造成对象不能被正常释放最终导致内存泄露的问题。weak 关键字的作用是弱引用，所引用对象的计数器不会加1，并在引用对象被释放的时候自动被设置为 nil。



下面的一段代码是在开发中常见的weak的使用

```objective-c
 NSObject *p = [[NSObject alloc] init];
 __weak NSObject *p1 = p;
```

此打断点跟踪汇编信息，可以发现底层库调了`objc_initWeak`函数

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2020-11-25-143524.jpg)



`objc_initWeak()` 这个方法。在进行编译过程前，clang 其实对 __weak 做了转换，将声明方式做出了如下调整。

```objective-c
NSObject objc_initWeak(&p, 对象指针);
```



## objc_initWeak()



其中的对象指针，就是代码中的 `[[NSObject alloc] init]` ，而 p 是我们传入的一个弱引用指针。而对于 `objc_initWeak()` 方法的实现，在 runtime 中的源码如下：

```c
id objc_initWeak(id *location, id newObj) {
    // 查看对象实例是否有效
    // 无效对象直接导致指针释放
    if (!newObj) {
        *location = nil;
        return nil;
    }

    // 这里传递了三个 bool 数值
    // 使用 template 进行常量参数传递是为了优化性能
    return storeWeak<false/*old*/, true/*new*/, true/*crash*/>
        (location, (objc_object*)newObj);
}
```

可以看出，这个函数仅仅是一个深层函数的调用入口，而一般的入口函数中，都会做一些简单的判断（例如 objc_msgSend 中的缓存判断），这里判断了其指针指向的类对象是否有效，无效直接释放，不再往深层调用函数。

需要注意的是，当修改弱引用的变量时，这个方法非线程安全。所以切记选择竞争带来的一些问题。

继续阅读 `objc_storeWeak()` 的实现：

```objective-c
// HaveOld:     true - 变量有值
//             false - 需要被及时清理，当前值可能为 nil
// HaveNew:     true - 需要被分配的新值，当前值可能为 nil
//             false - 不需要分配新值
// CrashIfDeallocating: true - 说明 newObj 已经释放或者 newObj 不支持弱引用，该过程需要暂停
//             false - 用 nil 替代存储
template <bool HaveOld, bool HaveNew, bool CrashIfDeallocating>
static id storeWeak(id *location, objc_object *newObj) {
    // 该过程用来更新弱引用指针的指向

    // 初始化 previouslyInitializedClass 指针
    Class previouslyInitializedClass = nil;
    id oldObj;

    // 声明两个 SideTable
    // ① 新旧散列创建
    SideTable *oldTable;
    SideTable *newTable;

    // 获得新值和旧值的锁存位置（用地址作为唯一标示）
    // 通过地址来建立索引标志，防止桶重复
    // 下面指向的操作会改变旧值
  retry:
    if (HaveOld) {
        // 更改指针，获得以 oldObj 为索引所存储的值地址
        oldObj = *location;
        oldTable = &SideTables()[oldObj];
    } else {
        oldTable = nil;
    }
    if (HaveNew) {
        // 更改新值指针，获得以 newObj 为索引所存储的值地址
        newTable = &SideTables()[newObj];
    } else {
        newTable = nil;
    }

    // 加锁操作，防止多线程中竞争冲突
    SideTable::lockTwo<HaveOld, HaveNew>(oldTable, newTable);

    // 避免线程冲突重处理
    // location 应该与 oldObj 保持一致，如果不同，说明当前的 location 已经处理过 oldObj 可是又被其他线程所修改
    if (HaveOld  &&  *location != oldObj) {
        SideTable::unlockTwo<HaveOld, HaveNew>(oldTable, newTable);
        goto retry;
    }

    // 防止弱引用间死锁
    // 并且通过 +initialize 初始化构造器保证所有弱引用的 isa 非空指向
    if (HaveNew  &&  newObj) {
        // 获得新对象的 isa 指针
        Class cls = newObj->getIsa();

        // 判断 isa 非空且已经初始化
        if (cls != previouslyInitializedClass  &&  
            !((objc_class *)cls)->isInitialized()) {
            // 解锁
            SideTable::unlockTwo<HaveOld, HaveNew>(oldTable, newTable);
            // 对其 isa 指针进行初始化
            _class_initialize(_class_getNonMetaClass(cls, (id)newObj));

            // 如果该类已经完成执行 +initialize 方法是最理想情况
            // 如果该类 +initialize 在线程中 
            // 例如 +initialize 正在调用 storeWeak 方法
            // 需要手动对其增加保护策略，并设置 previouslyInitializedClass 指针进行标记
            previouslyInitializedClass = cls;

            // 重新尝试
            goto retry;
        }
    }

    // ② 清除旧值
    if (HaveOld) {
        weak_unregister_no_lock(&oldTable->weak_table, oldObj, location);
    }

    // ③ 分配新值
    if (HaveNew) {
        newObj = (objc_object *)weak_register_no_lock(&newTable->weak_table, 
                                                      (id)newObj, location, 
                                                      CrashIfDeallocating);
        // 如果弱引用被释放 weak_register_no_lock 方法返回 nil 

        // 在引用计数表中设置若引用标记位
        if (newObj  &&  !newObj->isTaggedPointer()) {
            // 弱引用位初始化操作
            // 引用计数那张散列表的weak引用对象的引用计数中标识为weak引用
            newObj->setWeaklyReferenced_nolock();
        }

        // 之前不要设置 location 对象，这里需要更改指针指向
        *location = (id)newObj;
    }
    else {
        // 没有新值，则无需更改
    }

    SideTable::unlockTwo<HaveOld, HaveNew>(oldTable, newTable);

    return (id)newObj;
}
```

## 引用计数和弱引用依赖表 SideTable

`SideTable` 这个结构体，我给他起名**引用计数和弱引用依赖表**，因为它主要用于管理对象的引用计数和 `weak` 表。在 `NSObject.mm` 中声明其数据结构：

```c
struct SideTable {
    // 保证原子操作的自旋锁
    spinlock_t slock;
    // 引用计数的 hash 表
    RefcountMap refcnts;
    // weak 引用全局 hash 表
    weak_table_t weak_table;
}
```

在之前的 runtime 版本中，有一个较为重要的成员方法，用来根据对象的地址在缓存中取出对应的 `SideTable` 实例：

```c
static SideTable *tableForPointer(const void *p);
```

而在上面 `objc_storeWeak` 方法中，取出实例的方法变成了 `&SideTables()[xxxObj];` 这种方式。查看方法的实现，发现了如下函数：

```c
static StripedMap<SideTable>& SideTables() {
    return *reinterpret_cast<StripedMap<SideTable>*>(SideTableBuf);
}
```

在取出实例方法的实现中，使用了 C++ 标准转换运算符 **reinterpret_cast** ，其表达方式为：

```c
reinterpret_cast <new_type> (expression)
```

用来处理无关类型之间的转换。该关键字会产生一个新值，并**保证与原参数（expression）拥有完全相同的比特位**。

而 `StripedMap` 是一个模板类（Template Class），通过传入类（结构体）参数，会动态修改在该类中的一个 `array` 成员存储的元素类型，并且其中提供了一个针对于地址的 hash 算法，用作存储 key。可以说， `StripedMap` 提供了一套拥有将地址作为 key 的 hash table 解决方案，而该方案采用了模板类，是拥有泛型性的。

介绍了与对象相关联的 SideTable 检索方式，再来看 SideTable 的成员和作用。

对于 slock 和 refcnts 两个成员不用多说，第一个是为了防止竞争选择的自旋锁，第二个是协助对象的 isa 指针的 `extra_rc` 共同引用计数的变量（对于对象结果，在今后的文中提到）。这里主要看 `weak` 全局 hash 表的结构与作用。

```c
struct weak_table_t {
    // 保存了所有指向指定对象的 weak 指针
    weak_entry_t *weak_entries;
    // 存储空间
    size_t    num_entries;
    // 参与判断引用计数辅助量
    uintptr_t mask;
    // hash key 最大偏移值
    uintptr_t max_hash_displacement;
};
```

这是一个全局弱引用表。使用不定类型对象的地址作为 key ，用 weak_entry_t 类型结构体对象作为 value 。其中的 weak_entries 成员，从字面意思上看，即为弱引用表入口。其实现也是这样的。

```c
typedef objc_object ** weak_referrer_t;

struct weak_entry_t {
    DisguisedPtr<objc_object> referent;
    union {
        struct {
            weak_referrer_t *referrers;
            uintptr_t        out_of_line : 1;
            uintptr_t        num_refs : PTR_MINUS_1;
            uintptr_t        mask;
            uintptr_t        max_hash_displacement;
        };
        struct {
            // out_of_line=0 is LSB of one of these (don't care which)
            weak_referrer_t  inline_referrers[WEAK_INLINE_COUNT];
        };
 }
```

在 weak_entry_t 的结构中，`DisguisedPtr<objc_object> referent` 是对泛型对象的指针做了一个封装，通过这个泛型类来解决内存泄漏的问题。从注释中写 `out_of_line` 成员为最低有效位，当其为0的时候， `weak_referrer_t` 成员将扩展为多行静态 hash table。其实其中的 `weak_referrer_t` 是二维 `objc_object` 的别名，通过一个二维指针地址偏移，用下标作为 hash 的 key，做成了一个弱引用散列。

那么在有效位未生效的时候，`out_of_line` 、 `num_refs`、 `mask` 、 `max_hash_displacement` 有什么作用？以下是笔者自身的猜测：

- **out_of_line**：最低有效位，也是标志位。当标志位 0 时，增加引用表指针纬度。
- **num_refs**：引用数值。这里记录弱引用表中引用有效数字，因为弱引用表使用的是静态 hash 结构，所以需要使用变量来记录数目。
- **mask**：计数辅助量。
- **max_hash_displacement**：hash 元素上限阀值。

其实 out_of_line 的值通常情况下是等于零的，所以弱引用表总是一个 objc_objective 指针二维数组。一维 objc_objective 指针可构成一张弱引用散列表，通过第三纬度实现了多张散列表，并且表数量为 WEAK_INLINE_COUNT 。

总结一下 `StripedMap<SideTable>[]` ： `StripedMap` 是一个模板类，在这个类中有一个 array 成员，用来存储 PaddedT 对象，并且其中对于 `[]` 符的重载定义中，会返回这个 PaddedT 的 value 成员，这个 value 就是我们传入的 T 泛型成员，也就是 SideTable 对象。在 array 的下标中，这里使用了 indexForPointer 方法通过位运算计算下标，实现了静态的 Hash Table。而在 weak_table 中，其成员 weak_entry 会将传入对象的地址加以封装起来，并且其中也有访问全局弱引用表的入口。

## 旧对象解除注册操作 weak_unregister_no_lock

```c
#define WEAK_INLINE_COUNT 4

void weak_unregister_no_lock(weak_table_t *weak_table, id referent_id, 
                        id *referrer_id) {
    // 在入口方法中，传入了 weak_table 弱引用表，referent_id 旧对象以及 referent_id 旧对象对应的地址
    // 用指针去访问 oldObj 和 *location  
    objc_object *referent = (objc_object *)referent_id;
    objc_object **referrer = (objc_object **)referrer_id;

    weak_entry_t *entry;
    // 如果其对象为 nil，无需取消注册
    if (!referent) return;
    // weak_entry_for_referent 根据首对象查找 weak_entry
    if ((entry = weak_entry_for_referent(weak_table, referent))) {
        // 通过地址来解除引用关联    
        remove_referrer(entry, referrer);
        bool empty = true;
        // 检测 out_of_line 位的情况
        // 检测 num_refs 位的情况
        if (entry->out_of_line  &&  entry->num_refs != 0) {
            empty = false;
        }
        else {
            // 将引用表中记录为空
            for (size_t i = 0; i < WEAK_INLINE_COUNT; i++) {
                if (entry->inline_referrers[i]) {
                    empty = false; 
                    break;
                }
            }
        }
    // 从弱引用的 zone 表中删除
        if (empty) {
            weak_entry_remove(weak_table, entry);
        }
    }

    // 这里不会设置 *referrer = nil，因为 objc_storeWeak() 函数会需要该指针
}
```

该方法主要作用是将旧对象在 weak_table 中接触 weak 指针的对应绑定。根据函数名，称之为解除注册操作。从源码中，可以知道其功能就是从 weak_table 中接触 weak 指针的绑定。而其中的遍历查询，就是针对于 weak_entry 中的多张弱引用散列表。

## 新对象添加注册操作 weak_register_no_lock

```c
id weak_register_no_lock(weak_table_t *weak_table, id referent_id,
                      id *referrer_id, bool crashIfDeallocating) {
    // 在入口方法中，传入了 weak_table 弱引用表，referent_id 旧对象以及 referent_id 旧对象对应的地址
    // 用指针去访问 oldObj 和 *location
    objc_object *referent = (objc_object *)referent_id;
    objc_object **referrer = (objc_object **)referrer_id;

    // 检测对象是否生效、以及是否使用了 tagged pointer 技术
    if (!referent  ||  referent->isTaggedPointer()) return referent_id;

    // 保证引用对象是否有效
    // hasCustomRR 方法检查类（包括其父类）中是否含有默认的方法
    bool deallocating;
    if (!referent->ISA()->hasCustomRR()) {
        // 检查 dealloc 状态
        deallocating = referent->rootIsDeallocating();
    }
    else {
        // 会返回 referent 的 SEL_allowsWeakReference 方法的地址
        BOOL (*allowsWeakReference)(objc_object *, SEL) = 
            (BOOL(*)(objc_object *, SEL))
            object_getMethodImplementation((id)referent, 
                                           SEL_allowsWeakReference);
        if ((IMP)allowsWeakReference == _objc_msgForward) {
            return nil;
        }
        deallocating =
            ! (*allowsWeakReference)(referent, SEL_allowsWeakReference);
    }
    // 由于 dealloc 导致 crash ，并输出日志
    if (deallocating) {
        if (crashIfDeallocating) {
            _objc_fatal("Cannot form weak reference to instance (%p) of "
                        "class %s. It is possible that this object was "
                        "over-released, or is in the process of deallocation.",
                        (void*)referent, object_getClassName((id)referent));
        } else {
            return nil;
        }
    }

    // 记录并存储对应引用表 weak_entry
    weak_entry_t *entry;
    // 对于给定的弱引用查询 weak_table
    if ((entry = weak_entry_for_referent(weak_table, referent))) {
        // 增加弱引用表于附加对象上
        append_referrer(entry, referrer);
    } 
    else {
        // 自行创建弱引用表
        weak_entry_t new_entry;
        new_entry.referent = referent;
        new_entry.out_of_line = 0;
        new_entry.inline_referrers[0] = referrer;
        for (size_t i = 1; i < WEAK_INLINE_COUNT; i++) {
            new_entry.inline_referrers[i] = nil;
        }
        // 如果给定的弱引用表满容，进行自增长
        weak_grow_maybe(weak_table);
        // 向对象添加弱引用表关联，不进行检查直接修改指针指向
        weak_entry_insert(weak_table, &new_entry);
    }

    // 这里不会设置 *referrer = nil，因为 objc_storeWeak() 函数会需要该指针
    return referent_id;
}
```

这一步与上一步相反，通过 weak_register_no_lock 函数把心的对象进行注册操作，完成与对应的弱引用表进行绑定操作。



## 总结来说:

weaktable在每个sidetable中以结构体 `weak_entry_t` 存在,sidetable中储存着各种类对象,sidetable中包含了weaktable，rc引用计数器表,自选锁,当开发使用  `__weak typeof(self)weakSelf = self;` 时候, `weak_table_t` 保存了所有指向指定对象的 weak 指针,对象释放时，弱引用表置空

**1、weak的原理在于底层维护了一张weak_table_t结构的hash表，key是所指对象的地址，value是weak指针的地址数组。**

**2、weak 关键字的作用是弱引用，所引用对象的计数器不会加1，并在引用对象被释放的时候自动被设置为 nil。**

**3、对象释放时，调用`clearDeallocating`函数根据对象地址获取所有weak指针地址的数组，然后遍历这个数组把其中的数据设为nil，最后把这个entry从weak表中删除，最后清理对象的记录。**

**4、文章中介绍了SideTable、weak_table_t、weak_entry_t这样三个结构，它们之间的关系如下图所示。**

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2020-12-04-150728.jpg)


## Reference

[1.iOS底层原理：weak的实现原理](https://juejin.cn/post/6844904101839372295)

[2.weak 弱引用的实现方式](https://www.desgard.com/iOS-Source-Probe/Objective-C/Runtime/weak%20%E5%BC%B1%E5%BC%95%E7%94%A8%E7%9A%84%E5%AE%9E%E7%8E%B0%E6%96%B9%E5%BC%8F.html)