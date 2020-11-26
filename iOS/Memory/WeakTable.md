# weak的实现原理

> 使用场景都比较清晰，避免出现对象之间的强强引用而造成对象不能被正常释放最终导致内存泄露的问题。weak 关键字的作用是弱引用，所引用对象的计数器不会加1，并在引用对象被释放的时候自动被设置为 nil。



下面的一段代码是在开发中常见的weak的使用

```objective-c
Person *object = [[Person alloc]init];
id __weak objc = object;
```

此打断点跟踪汇编信息，可以发现底层库调了`objc_initWeak`函数

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2020-11-25-143524.jpg)



## 1.objc_initWeak方法

下是`objc_initWeak`方法的底层源码

```objective-c
id objc_initWeak(id *location, id newObj)
{
    if (!newObj) {
        *location = nil;
        return nil;
    }

    return storeWeak<DontHaveOld, DoHaveNew, DoCrashIfDeallocating>
        (location, (objc_object*)newObj);
}

```

该方法的两个参数`location`和`newObj`。

> - **location** ：**`__weak指针`的地址**，存储指针的地址，这样便可以在最后将其指向的对象置为nil。
> - **newObj** ：所引用的对象。即例子中的obj 。

`objc_initWeak`方法只是一个深层次函数调用的入口，在该方法内部调用了`storeWeak`方法。



## 2.storeWeak方法

```objective-c
storeWeak(id *location, objc_object *newObj)
{
    assert(haveOld  ||  haveNew);
    if (!haveNew) assert(newObj == nil);

    Class previouslyInitializedClass = nil;
    id oldObj;
    SideTable *oldTable;
    SideTable *newTable;

    // Acquire locks for old and new values.
    // Order by lock address to prevent lock ordering problems.
    // Retry if the old value changes underneath us.
 retry:
    if (haveOld) { // 如果weak ptr之前弱引用过一个obj，则将这个obj所对应的SideTable取出，赋值给oldTable
        oldObj = *location;
        oldTable = &SideTables()[oldObj];
    } else {
        oldTable = nil; // 如果weak ptr之前没有弱引用过一个obj，则oldTable = nil
    }
    if (haveNew) { // 如果weak ptr要weak引用一个新的obj，则将该obj对应的SideTable取出，赋值给newTable
        newTable = &SideTables()[newObj];
    } else {
        newTable = nil; // 如果weak ptr不需要引用一个新obj，则newTable = nil
    }
    
    // 加锁操作，防止多线程中竞争冲突
    SideTable::lockTwo<haveOld, haveNew>(oldTable, newTable);

    // location 应该与 oldObj 保持一致，如果不同，说明当前的 location 已经处理过 oldObj 可是又被其他线程所修改
    if (haveOld  &&  *location != oldObj) {
        SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);
        goto retry;
    }

    // Prevent a deadlock between the weak reference machinery
    // and the +initialize machinery by ensuring that no
    // weakly-referenced object has an un-+initialized isa.
    if (haveNew  &&  newObj) {
        Class cls = newObj->getIsa();
        if (cls != previouslyInitializedClass  &&
            !((objc_class *)cls)->isInitialized())  // 如果cls还没有初始化，先初始化，再尝试设置weak
        {
            SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);
            _class_initialize(_class_getNonMetaClass(cls, (id)newObj));

            // If this class is finished with +initialize then we're good.
            // If this class is still running +initialize on this thread
            // (i.e. +initialize called storeWeak on an instance of itself)
            // then we may proceed but it will appear initializing and
            // not yet initialized to the check above.
            // Instead set previouslyInitializedClass to recognize it on retry.
            previouslyInitializedClass = cls; // 这里记录一下previouslyInitializedClass， 防止改if分支再次进入

            goto retry; // 重新获取一遍newObj，这时的newObj应该已经初始化过了
        }
    }

    // Clean up old value, if any.
    if (haveOld) {
        weak_unregister_no_lock(&oldTable->weak_table, oldObj, location); // 如果weak_ptr之前弱引用过别的对象oldObj，则调用weak_unregister_no_lock，在oldObj的weak_entry_t中移除该weak_ptr地址
    }

    // Assign new value, if any.
    if (haveNew) { // 如果weak_ptr需要弱引用新的对象newObj
        // (1) 调用weak_register_no_lock方法，将weak ptr的地址记录到newObj对应的weak_entry_t中
        newObj = (objc_object *)
            weak_register_no_lock(&newTable->weak_table, (id)newObj, location,
                                  crashIfDeallocating);
        // weak_register_no_lock returns nil if weak store should be rejected
        
        // (2) 更新newObj的isa的weakly_referenced bit标志位
        // Set is-weakly-referenced bit in refcount table.
        if (newObj  &&  !newObj->isTaggedPointer()) {
            newObj->setWeaklyReferenced_nolock();
        }

        // Do not set *location anywhere else. That would introduce a race.
        // （3）*location 赋值，也就是将weak ptr直接指向了newObj。可以看到，这里并没有将newObj的引用计数+1
        *location = (id)newObj; // 将weak ptr指向object
    }
    else {
        // No new value. The storage is not changed.
    }
    
    // 解锁，其他线程可以访问oldTable, newTable了
    SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);

    return (id)newObj; // 返回newObj，此时的newObj与刚传入时相比，weakly-referenced bit位置1
}

```



`storeWeak`方法实际上是接收了5个参数，分别是`haveOld、haveNew和crashIfDeallocating`，这三个参数都是以模板的方式传入的，是三个bool类型的参数。 分别表示weak指针之前是否指向了一个弱引用，weak指针是否需要指向一个新的引用，若果被弱引用的对象正在析构，此时再弱引用该对象是否应该crash。

该方法维护了`oldTable`和`newTable`分别表示旧的引用弱表和新的弱引用表，它们都是`SideTable`的hash表。

如果weak指针之前指向了一个弱引用，则会调用`weak_unregister_no_lock`方法将旧的weak指针地址移除。

如果weak指针需要指向一个新的引用，则会调用`weak_register_no_lock`方法将新的weak指针地址添加到弱引用表中。

调用`setWeaklyReferenced_nolock`方法修改weak新引用的对象的bit标志位



# 3、SideTable

先来看下SideTable的定义。

```
struct SideTable {
    spinlock_t slock;
    RefcountMap refcnts;
    weak_table_t weak_table;
}
复制代码
```

SideTable的定义很清晰，有三个成员:

> - **spinlock_t slock** : 自旋锁，用于上锁/解锁 SideTable。
> - **RefcountMap refcnts** ：用来存储OC对象的引用计数的 `hash表`(仅在未开启isa优化或在isa优化情况下isa_t的引用计数溢出时才会用到)。
> - **weak_table_t weak_table** : 存储对象弱引用指针的`hash表`。是OC中weak功能实现的核心数据结构。

### 3.1、weak_table_t

先来看下`weak_table_t`的底层代码。

```
struct weak_table_t {
    weak_entry_t *weak_entries;
    size_t    num_entries;
    uintptr_t mask;
    uintptr_t max_hash_displacement;
};
复制代码
```

> - **weak_entries**： hash数组，用来存储弱引用对象的相关信息weak_entry_t
> - **num_entries**：  hash数组中的元素个数
> - **mask**：hash数组长度-1，会参与hash计算。（注意，这里是hash数组的长度，而不是元素个数。比如，数组长度可能是64，而元素个数仅存了2个）
> - **max_hash_displacement**：可能会发生的hash冲突的最大次数，用于判断是否出现了逻辑错误（hash表中的冲突次数绝不会超过改值）

`weak_table_t`是一个典型的hash结构。`weak_entries`是一个动态数组，用来存储`weak_entry_t`类型的元素，这些元素实际上就是OC对象的弱引用信息。

### 3.2、weak_entry_t

`weak_entry_t`的结构也是一个hash结构，其存储的元素是弱引用对象指针的指针， 通过操作指针的指针，就可以使得weak 引用的指针在对象析构后，指向nil。

```
#define WEAK_INLINE_COUNT 4
#define REFERRERS_OUT_OF_LINE 2

struct weak_entry_t {
    DisguisedPtr<objc_object> referent; // 被弱引用的对象
    
    // 引用该对象的对象列表，联合。 引用个数小于4，用inline_referrers数组。 用个数大于4，用动态数组weak_referrer_t *referrers
    union {
        struct {
            weak_referrer_t *referrers;                      // 弱引用该对象的对象指针地址的hash数组
            uintptr_t        out_of_line_ness : 2;           // 是否使用动态hash数组标记位
            uintptr_t        num_refs : PTR_MINUS_2;         // hash数组中的元素个数
            uintptr_t        mask;                           // hash数组长度-1，会参与hash计算。（注意，这里是hash数组的长度，而不是元素个数。比如，数组长度可能是64，而元素个数仅存了2个）素个数）。
            uintptr_t        max_hash_displacement;          // 可能会发生的hash冲突的最大次数，用于判断是否出现了逻辑错误（hash表中的冲突次数绝不会超过改值）
        };
        struct {
            // out_of_line_ness field is low bits of inline_referrers[1]
            weak_referrer_t  inline_referrers[WEAK_INLINE_COUNT];
        };
    };

    bool out_of_line() {
        return (out_of_line_ness == REFERRERS_OUT_OF_LINE);
    }

    weak_entry_t& operator=(const weak_entry_t& other) {
        memcpy(this, &other, sizeof(other));
        return *this;
    }

    weak_entry_t(objc_object *newReferent, objc_object **newReferrer)
        : referent(newReferent) // 构造方法，里面初始化了静态数组
    {
        inline_referrers[0] = newReferrer;
        for (int i = 1; i < WEAK_INLINE_COUNT; i++) {
            inline_referrers[i] = nil;
        }
    }
};
复制代码
```

可以看到在`weak_entry_t`的结构定义中有联合体，在联合体的内部有定长数组`inline_referrers[WEAK_INLINE_COUNT]`和动态数组`weak_referrer_t *referrers`两种方式来存储弱引用对象的指针地址。通过`out_of_line()`这样一个函数方法来判断采用哪种存储方式。当弱引用该对象的指针数目小于等于`WEAK_INLINE_COUNT`时，使用定长数组。当超过`WEAK_INLINE_COUNT`时，会将定长数组中的元素转移到动态数组中，并之后都是用动态数组存储。

到这里我们已经清楚了**弱引用表的结构是一个hash结构的表，Key是所指对象的地址，Value是weak指针的地址（这个地址的值是所指对象的地址）数组**。那么接下来看看这个弱引用表是怎么维护这些数据的。

# 4、weak_register_no_lock方法添加弱引用

```
id 
weak_register_no_lock(weak_table_t *weak_table, id referent_id, 
                      id *referrer_id, bool crashIfDeallocating)
{
    objc_object *referent = (objc_object *)referent_id;
    objc_object **referrer = (objc_object **)referrer_id;

    // 如果referent为nil 或 referent 采用了TaggedPointer计数方式，直接返回，不做任何操作
    if (!referent  ||  referent->isTaggedPointer()) return referent_id;

    // 确保被引用的对象可用（没有在析构，同时应该支持weak引用）
    bool deallocating;
    if (!referent->ISA()->hasCustomRR()) {
        deallocating = referent->rootIsDeallocating();
    }
    else {
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
    // 正在析构的对象，不能够被弱引用
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

    // now remember it and where it is being stored
    // 在 weak_table中找到referent对应的weak_entry,并将referrer加入到weak_entry中
    weak_entry_t *entry;
    if ((entry = weak_entry_for_referent(weak_table, referent))) { // 如果能找到weak_entry,则讲referrer插入到weak_entry中
        append_referrer(entry, referrer); 	// 将referrer插入到weak_entry_t的引用数组中
    } 
    else { // 如果找不到，就新建一个
        weak_entry_t new_entry(referent, referrer);  
        weak_grow_maybe(weak_table);
        weak_entry_insert(weak_table, &new_entry);
    }

    // Do not set *referrer. objc_storeWeak() requires that the 
    // value not change.

    return referent_id;
}
复制代码
```

这个方法需要传入四个参数，它们代表的意义如下：

> - **weak_table**：`weak_table_t`结构类型的全局的弱引用表。
> - **referent_id**：weak指针。
> - ***referrer_id**：weak指针地址。
> - **crashIfDeallocating** ：若果被弱引用的对象正在析构，此时再弱引用该对象是否应该crash。

从上面的代码我么可以知道该方法主要的做了如下几个方便的工作。

> 1. 如果referent为nil 或 referent 采用了`TaggedPointer`计数方式，直接返回，不做任何操作。
> 2. 如果对象正在析构，则抛出异常。
> 3. 如果对象不能被weak引用，直接返回nil。
> 4. 如果对象没有再析构且可以被weak引用，则调用`weak_entry_for_referent`方法根据弱引用对象的地址从弱引用表中找到对应的weak_entry，如果能够找到则调用`append_referrer`方法向其中插入weak指针地址。否则新建一个weak_entry。

### 4.1、weak_entry_for_referent取元素

```
static weak_entry_t *
weak_entry_for_referent(weak_table_t *weak_table, objc_object *referent)
{
    assert(referent);

    weak_entry_t *weak_entries = weak_table->weak_entries;

    if (!weak_entries) return nil;

    size_t begin = hash_pointer(referent) & weak_table->mask;  // 这里通过 & weak_table->mask的位操作，来确保index不会越界
    size_t index = begin;
    size_t hash_displacement = 0;
    while (weak_table->weak_entries[index].referent != referent) {
        index = (index+1) & weak_table->mask;
        if (index == begin) bad_weak_table(weak_table->weak_entries); // 触发bad weak table crash
        hash_displacement++;
        if (hash_displacement > weak_table->max_hash_displacement) { // 当hash冲突超过了可能的max hash 冲突时，说明元素没有在hash表中，返回nil 
            return nil;
        }
    }
    
    return &weak_table->weak_entries[index];
}
复制代码
```

### 4.2、append_referrer添加元素

```
static void append_referrer(weak_entry_t *entry, objc_object **new_referrer)
{
    if (! entry->out_of_line()) { // 如果weak_entry 尚未使用动态数组，走这里
        // Try to insert inline.
        for (size_t i = 0; i < WEAK_INLINE_COUNT; i++) {
            if (entry->inline_referrers[i] == nil) {
                entry->inline_referrers[i] = new_referrer;
                return;
            }
        }
        
        // 如果inline_referrers的位置已经存满了，则要转型为referrers，做动态数组。
        // Couldn't insert inline. Allocate out of line.
        weak_referrer_t *new_referrers = (weak_referrer_t *)
            calloc(WEAK_INLINE_COUNT, sizeof(weak_referrer_t));
        // This constructed table is invalid, but grow_refs_and_insert
        // will fix it and rehash it.
        for (size_t i = 0; i < WEAK_INLINE_COUNT; i++) {
            new_referrers[i] = entry->inline_referrers[I];
        }
        entry->referrers = new_referrers;
        entry->num_refs = WEAK_INLINE_COUNT;
        entry->out_of_line_ness = REFERRERS_OUT_OF_LINE;
        entry->mask = WEAK_INLINE_COUNT-1;
        entry->max_hash_displacement = 0;
    }

    // 对于动态数组的附加处理：
    assert(entry->out_of_line()); // 断言： 此时一定使用的动态数组

    if (entry->num_refs >= TABLE_SIZE(entry) * 3/4) { // 如果动态数组中元素个数大于或等于数组位置总空间的3/4，则扩展数组空间为当前长度的一倍
        return grow_refs_and_insert(entry, new_referrer); // 扩容，并插入
    }
    
    // 如果不需要扩容，直接插入到weak_entry中
    // 注意，weak_entry是一个哈希表，key：w_hash_pointer(new_referrer) value: new_referrer
    
    // 细心的人可能注意到了，这里weak_entry_t 的hash算法和 weak_table_t的hash算法是一样的，同时扩容/减容的算法也是一样的
    size_t begin = w_hash_pointer(new_referrer) & (entry->mask); // '& (entry->mask)' 确保了 begin的位置只能大于或等于 数组的长度
    size_t index = begin;  // 初始的hash index
    size_t hash_displacement = 0;  // 用于记录hash冲突的次数，也就是hash再位移的次数
    while (entry->referrers[index] != nil) {
        hash_displacement++;
        index = (index+1) & entry->mask;  // index + 1, 移到下一个位置，再试一次能否插入。（这里要考虑到entry->mask取值，一定是：0x111, 0x1111, 0x11111, ... ，因为数组每次都是*2增长，即8， 16， 32，对应动态数组空间长度-1的mask，也就是前面的取值。）
        if (index == begin) bad_weak_table(entry); // index == begin 意味着数组绕了一圈都没有找到合适位置，这时候一定是出了什么问题。
    }
    if (hash_displacement > entry->max_hash_displacement) { // 记录最大的hash冲突次数, max_hash_displacement意味着: 我们尝试至多max_hash_displacement次，肯定能够找到object对应的hash位置
        entry->max_hash_displacement = hash_displacement;
    }
    // 将ref存入hash数组，同时，更新元素个数num_refs
    weak_referrer_t &ref = entry->referrers[index];
    ref = new_referrer;
    entry->num_refs++;
}
复制代码
```

这段代码首先是使用定长数组还是动态数组，如果是使用定长数组，则直接weak指针地址直接添加到数组即可，如果定长数组已经用尽，则需要将定长数组中的元素转存到动态数组中。

# 


作者：夜幕降临耶
链接：https://juejin.cn/post/6844904101839372295
来源：掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

## Reference

[1.iOS底层原理：weak的实现原理](https://juejin.cn/post/6844904101839372295)

