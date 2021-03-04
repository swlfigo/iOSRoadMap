# Lock 

## 锁

#### 1.线程安全

当一个线程访问数据的时候，其他的线程不能对其进行访问，直到该线程访问完毕。简单来讲就是在同一时刻，对同一个数据操作的线程只有一个。而线程不安全，则是在同一时刻可以有多个线程对该数据进行访问，从而得不到预期的结果

**即线程内操作了一个线程外的非线程安全变量，这个时候一定要考虑线程安全和同步**

#### 2.检测安全



![img](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-03-03-065714.jpg)



#### 3.锁的作用

`锁`作为一种非强制的机制，被用来保证线程安全。每一个线程在访问数据或者资源前，要先获取（`Acquire`）锁，并在访问结束之后释放（`Release`）锁。如果锁已经被占用，其它试图获取锁的线程会等待，直到锁重新可用

**注：不要将过多的其他操作代码放到锁里面，否则一个线程执行的时候另一个线程就一直在等待，就无法发挥多线程的作用了**

#### 4.锁的分类

在iOS中锁的基本种类只有两种：`互斥锁`、`自旋锁`，其他的比如`条件锁`、`递归锁`、`信号量`都是上层的封装和实现

#### 5. 互斥锁

`互斥锁`(Mutual exclusion，缩写`Mutex`)防止两条线程同时对同一公共资源(比如全局变量)进行读写的机制。当获取锁操作失败时，线程会进入睡眠，等待锁释放时被唤醒

`互斥锁`又分为：

- `递归锁`：可重入锁，同一个线程在锁释放前可再次获取锁，即可以递归调用
- `非递归锁`：不可重入，必须等锁释放后才能再次获取锁

#### 6. 自旋锁

`自旋锁`：线程反复检查锁变量是否可⽤。由于线程在这⼀过程中保持执⾏， 因此是⼀种`忙等待`。⼀旦获取了⾃旋锁，线程会⼀直保持该锁，直⾄显式释 放⾃旋锁

`⾃旋锁`避免了进程上下⽂的调度开销，因此对于线程只会阻塞很短时间的场合是有效的

#### 7.互斥锁和自旋锁的区别

- `互斥锁`在线程获取锁但没有获取到时，线程会进入休眠状态，等锁被释放时线程会被唤醒
- `自旋锁`的线程则会一直处于等待状态（忙等待）不会进入休眠——因此效率高



## 二、自旋锁

#### 1.OSSpinLock

自从`OSSpinLock`出现了安全问题之后就废弃了。自旋锁之所以不安全，是因为自旋锁由于获取锁时，线程会一直处于忙等待状态，造成了任务的优先级反转

而`OSSpinLock`忙等的机制就可能造成高优先级一直`running等待`，占用CPU时间片；而低优先级任务无法抢占时间片，变成迟迟完不成，不释放锁的情况

#### 2.atomic

###### 2.1 atomic原理

在[iOS探索 KVC原理及自定义](https://juejin.im/post/6844904086744104968#heading-3)中有提到自动生成的setter方法会根据修饰符不同调用不同方法，最后统一调用`reallySetProperty`方法，其中就有一段关于`atomic`修饰词的代码

```objective-c
static inline void reallySetProperty(id self, SEL _cmd, id newValue, ptrdiff_t offset, bool atomic, bool copy, bool mutableCopy)
{
    if (offset == 0) {
        object_setClass(self, newValue);
        return;
    }

    id oldValue;
    id *slot = (id*) ((char*)self + offset);

    if (copy) {
        newValue = [newValue copyWithZone:nil];
    } else if (mutableCopy) {
        newValue = [newValue mutableCopyWithZone:nil];
    } else {
        if (*slot == newValue) return;
        newValue = objc_retain(newValue);
    }

    if (!atomic) {
        oldValue = *slot;
        *slot = newValue;
    } else {
        spinlock_t& slotlock = PropertyLocks[slot];
        slotlock.lock();
        oldValue = *slot;
        *slot = newValue;        
        slotlock.unlock();
    }

    objc_release(oldValue);
}
```

比对一下`atomic`的逻辑分支：

- 原子性修饰的属性进行了`spinlock`加锁处理
- 非原子性的属性除了没加锁，其他逻辑与`atomic`一般无二

等等，前面不是刚说`OSSpinLock`因为安全问题被废弃了吗，但是苹果源码怎么还在使用呢？其实点进去就会发现用`os_unfair_lock`替代了`OSSpinLock`（iOS10之后替换）

```objective-c
using spinlock_t = mutex_tt<LOCKDEBUG>;

class mutex_tt : nocopy_t {
    os_unfair_lock mLock;
    ...
}
```

> 同时为了哈希不冲突，还使用`加盐操作`进行加锁

`getter`方法亦是如此：atomic修饰的属性进行加锁处理

```objective-c
id objc_getProperty(id self, SEL _cmd, ptrdiff_t offset, BOOL atomic) {
    if (offset == 0) {
        return object_getClass(self);
    }

    // Retain release world
    id *slot = (id*) ((char*)self + offset);
    if (!atomic) return *slot;
        
    // Atomic retain release world
    spinlock_t& slotlock = PropertyLocks[slot];
    slotlock.lock();
    id value = objc_retain(*slot);
    slotlock.unlock();
    
    // for performance, we (safely) issue the autorelease OUTSIDE of the spinlock.
    return objc_autoreleaseReturnValue(value);
}
```

###### 2.2 atomic修饰的属性绝对安全吗？

`atomic`只能保证setter、getter方法的线程安全，并不能保证数据安全

![img](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-03-04-083856.jpg)

如上图所示，被`atomic`修饰的`index变量`分别在两次并发异步for循环`10000次`后输出的结果并不等于`20000`。由此可以得出结论：



- `atomic`保证变量在取值和赋值时的线程安全
- 但不能保证`self.index+1`也是安全的
- 如果改成`self.index=i`是能保证setter方法的线程安全的



#### 3. 读写锁

`读写锁`实际是一种特殊的`自旋锁`，它把对共享资源的访问者划分成读者和写者，读者只对共享资源进行读访问，写者则需要对共享资源进行写操作。这种锁相对于`自旋锁`而言，能提高并发性，因为在多处理器系统中，它允许同时有多个读者来访问共享资源，最大可能的读者数为实际的`CPU`数

- 写者是排他性的，⼀个读写锁同时只能有⼀个写者或多个读者（与CPU数相关），但不能同时既有读者⼜有写者。在读写锁保持期间也是抢占失效的
- 如果读写锁当前没有读者，也没有写者，那么写者可以⽴刻获得读写锁，否则它必须⾃旋在那⾥，直到没有任何写者或读者。如果读写锁没有写者，那么读者可以⽴即获得该读写锁，否则读者必须⾃旋在那⾥，直到写者释放该读写锁

```
// 导入头文件
#import <pthread.h>
// 全局声明读写锁
pthread_rwlock_t lock;
// 初始化读写锁
pthread_rwlock_init(&lock, NULL);
// 读操作-加锁
pthread_rwlock_rdlock(&lock);
// 读操作-尝试加锁
pthread_rwlock_tryrdlock(&lock);
// 写操作-加锁
pthread_rwlock_wrlock(&lock);
// 写操作-尝试加锁
pthread_rwlock_trywrlock(&lock);
// 解锁
pthread_rwlock_unlock(&lock);
// 释放锁
pthread_rwlock_destroy(&lock);
复制代码
```

平时很少会直接使用读写锁`pthread_rwlock_t`，更多的是采用其他方式，例如使用[栅栏函数](https://juejin.im/post/6844904138623418376#heading-6)完成读写锁的需求

## 三、互斥锁

#### 1.pthread_mutex

`pthread_mutex`就是`互斥锁`本身——当锁被占用，而其他线程申请锁时，不是使用忙等，而是阻塞线程并睡眠

使用如下：

```objective-c
// 导入头文件
#import <pthread.h>
// 全局声明互斥锁
pthread_mutex_t _lock;
// 初始化互斥锁
pthread_mutex_init(&_lock, NULL);
// 加锁
pthread_mutex_lock(&_lock);
// 这里做需要线程安全操作
// ...
// 解锁 
pthread_mutex_unlock(&_lock);
// 释放锁
pthread_mutex_destroy(&_lock);

```

[YYKit的YYMemoryCach](https://github.com/ibireme/YYKit/blob/3869686e0e560db0b27a7140188fad771e271508/YYKit/Cache/YYMemoryCache.m)有使用到`pthread_mutex`

#### 2.@synchronized

`@synchronized`可能是日常开发中用的比较多的一种互斥锁，因为它的使用比较简单，但并不是在任意场景下都能使用`@synchronized`，且它的性能较低

```objective-c
@synchronized (obj) {}
```

接下来就通过源码探索来看一下`@synchronized`在使用中的注意事项

- 通过**汇编**能发现`@synchronized`就是实现了`objc_sync_enter`和 `objc_sync_exit`两个方法
- 通过**符号断点**能知道这两个方法都是在`objc源码`中的
- 通过**clang**也能得到一些信息：

```objective-c
int main(int argc, char * argv[]) {
    NSString * appDelegateClassName;
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 

        appDelegateClassName = NSStringFromClass(((Class (*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("AppDelegate"), sel_registerName("class")));
        {
            id _rethrow = 0;
            id _sync_obj = (id)appDelegateClassName;
            objc_sync_enter(_sync_obj);
            try {
                struct _SYNC_EXIT {
                    _SYNC_EXIT(id arg) : sync_exit(arg) {}
                    ~_SYNC_EXIT() {
                        objc_sync_exit(sync_exit);
                    }
                    id sync_exit;
                }
                _sync_exit(_sync_obj);
            }
            catch (id e) {_rethrow = e;}
            {
                struct _FIN { _FIN(id reth) : rethrow(reth) {}
                    ~_FIN() { if (rethrow) objc_exception_throw(rethrow); }
                    id rethrow;
                }_fin_force_rethow(_rethrow);
            }
        }
    }
    return UIApplicationMain(argc, argv, __null, appDelegateClassName);
}
```

###### 2.1 源码分析

在`objc源码`中找到`objc_sync_enter`和`objc_sync_exit`

```objective-c
// Begin synchronizing on 'obj'. 
// Allocates recursive mutex associated with 'obj' if needed.
// Returns OBJC_SYNC_SUCCESS once lock is acquired.  
int objc_sync_enter(id obj)
{
    int result = OBJC_SYNC_SUCCESS;

    if (obj) {
        SyncData* data = id2data(obj, ACQUIRE);
        assert(data);
        data->mutex.lock();
    } else {
        // @synchronized(nil) does nothing
        if (DebugNilSync) {
            _objc_inform("NIL SYNC DEBUG: @synchronized(nil); set a breakpoint on objc_sync_nil to debug");
        }
        objc_sync_nil();
    }

    return result;
}

// End synchronizing on 'obj'. 
// Returns OBJC_SYNC_SUCCESS or OBJC_SYNC_NOT_OWNING_THREAD_ERROR
int objc_sync_exit(id obj)
{
    int result = OBJC_SYNC_SUCCESS;
    
    if (obj) {
        SyncData* data = id2data(obj, RELEASE); 
        if (!data) {
            result = OBJC_SYNC_NOT_OWNING_THREAD_ERROR;
        } else {
            bool okay = data->mutex.tryUnlock();
            if (!okay) {
                result = OBJC_SYNC_NOT_OWNING_THREAD_ERROR;
            }
        }
    } else {
        // @synchronized(nil) does nothing
    }
	
    return result;
}
```

1. 首先从它的注释中`recursive mutex`可以得出`@synchronized`是递归锁
2. 如果锁的对象`obj`不存在时分别会走`objc_sync_nil()`和`不做任何操作`（源码分析可以先解决简单的逻辑分支）

```objective-c
BREAKPOINT_FUNCTION(
    void objc_sync_nil(void)
);
```

这也是`@synchronized`作为递归锁但能防止死锁的原因所在：在不断递归的过程中如果对象不存在了就会停止递归从而防止死锁

1. 正常情况下（obj存在）会通过`id2data`方法生成一个`SyncData`对象

- `nextData`指的是链表中下一个SyncData
- `object`指的是当前加锁的对象
- `threadCount`表示使用该对象进行加锁的线程数
- `mutex`即对象所关联的锁

```objective-c
typedef struct alignas(CacheLineSize) SyncData {
    struct SyncData* nextData;
    DisguisedPtr<objc_object> object;
    int32_t threadCount;  // number of THREADS using this block
    recursive_mutex_t mutex;
} SyncData;
```

###### 2.2 准备SyncData

```objective-c
static SyncData* id2data(id object, enum usage why)
{
    spinlock_t *lockp = &LOCK_FOR_OBJ(object);
    SyncData **listp = &LIST_FOR_OBJ(object);
    SyncData* result = NULL;
    ...
}
```

`id2data`先将返回对象`SyncData类型的result`准备好，后续进行数据填充

```objective-c
#define LOCK_FOR_OBJ(obj) sDataLists[obj].lock
#define LIST_FOR_OBJ(obj) sDataLists[obj].data

static StripedMap<SyncList> sDataLists;

struct SyncList {
    SyncData *data;
    spinlock_t lock;

    constexpr SyncList() : data(nil), lock(fork_unsafe_lock) { }
};
```

其中通过两个宏定义去取得`SyncList`中的`data`和`lock`——`static StripedMap<SyncList> sDataLists` 可以理解成 `NSArray<id> list`

既然`@synchronized`能在任意地方（VC、View、Model等）使用，那么底层必然维护着一张全局的表（类似于weak表）。而从`SyncList`和`SyncData`的结构可以证实系统确实在底层维护着一张哈希表，里面存储着`SyncList结构`的数据。`SyncList`和`SyncData`的关系如下图所示：

![img](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-03-04-092008.jpg)



###### 2.3 使用快速缓存

```objective-c
static SyncData* id2data(id object, enum usage why)
{
    ...
#if SUPPORT_DIRECT_THREAD_KEYS
    // Check per-thread single-entry fast cache for matching object
    // 检查每线程单项快速缓存中是否有匹配的对象
    bool fastCacheOccupied = NO;
    SyncData *data = (SyncData *)tls_get_direct(SYNC_DATA_DIRECT_KEY);
    if (data) {
        fastCacheOccupied = YES;

        if (data->object == object) {
            // Found a match in fast cache.
            uintptr_t lockCount;

            result = data;
            lockCount = (uintptr_t)tls_get_direct(SYNC_COUNT_DIRECT_KEY);
            if (result->threadCount <= 0  ||  lockCount <= 0) {
                _objc_fatal("id2data fastcache is buggy");
            }

            switch(why) {
            case ACQUIRE: {
                lockCount++;
                tls_set_direct(SYNC_COUNT_DIRECT_KEY, (void*)lockCount);
                break;
            }
            case RELEASE:
                lockCount--;
                tls_set_direct(SYNC_COUNT_DIRECT_KEY, (void*)lockCount);
                if (lockCount == 0) {
                    // remove from fast cache
                    tls_set_direct(SYNC_DATA_DIRECT_KEY, NULL);
                    // atomic because may collide with concurrent ACQUIRE
                    OSAtomicDecrement32Barrier(&result->threadCount);
                }
                break;
            case CHECK:
                // do nothing
                break;
            }

            return result;
        }
    }
#endif
    ...
}
```

这里有个重要的知识点——`TLS`：`TLS`全称为`Thread Local Storage`，在iOS中每个线程都拥有自己的`TLS`，负责保存本线程的一些变量， 且`TLS`无需锁保护, `快速缓存`的含义为：定义两个变量`SYNC_DATA_DIRECT_KEY/SYNC_COUNT_DIRECT_KEY`，与`tsl_get_direct/tls_set_direct`配合可以从线程局部缓存中快速取得`SyncCacheItem.data`和`SyncCacheItem.lockCount

如果在缓存中找到当前对象，就拿出当前被锁的次数`lockCount`，再根据传入参数类型(获取、释放、查看)对`lockCount`分别进行操作

- 获取资源`ACQUIRE`：`lockCount++`并根据`key`值存入被锁次数
- 释放资源`RELEASE`：`lockCount++`并根据`key`值存入被锁次数。如果次数变为0，此时锁也不复存在，需要从快速缓存移除并清空线程数`threadCount`
- 查看资源`check`：不操作

> lockCount表示被锁的次数，意味着能多次进入，从侧面表现出了递归性

###### 2.4 获取该线程下的SyncCache

这个逻辑分支是找不到确切的线程标记只能进行所有的缓存遍历

```objective-c
static SyncData* id2data(id object, enum usage why)
{
    ...
    SyncCache *cache = fetch_cache(NO);
    if (cache) {
        unsigned int i;
        for (i = 0; i < cache->used; i++) {
            SyncCacheItem *item = &cache->list[i];
            if (item->data->object != object) continue;

            // Found a match.
            result = item->data;
            if (result->threadCount <= 0  ||  item->lockCount <= 0) {
                _objc_fatal("id2data cache is buggy");
            }
                
            switch(why) {
            case ACQUIRE:
                item->lockCount++;
                break;
            case RELEASE:
                item->lockCount--;
                if (item->lockCount == 0) {
                    // remove from per-thread cache
                    cache->list[i] = cache->list[--cache->used];
                    // atomic because may collide with concurrent ACQUIRE
                    OSAtomicDecrement32Barrier(&result->threadCount);
                }
                break;
            case CHECK:
                // do nothing
                break;
            }

            return result;
        }
    }
    ...
}
```

这里介绍一下`SyncCache`和`SyncCacheItem`

```objective-c
typedef struct {
    SyncData *data;             //该缓存条目对应的SyncData
    unsigned int lockCount;     //该对象在该线程中被加锁的次数
} SyncCacheItem;

typedef struct SyncCache {
    unsigned int allocated;     //该缓存此时对应的缓存大小
    unsigned int used;          //该缓存此时对应的已使用缓存大小
    SyncCacheItem list[0];      //SyncCacheItem数组
} SyncCache;
```

- `SyncCacheItem`用来记录某个`SyncData`在某个线程中被加锁的记录，一个`SyncData`可以被多个`SyncCacheItem`持有
- `SyncCache`用来记录某个线程中所有`SyncCacheItem`，并且记录了缓存大小以及已使用缓存大小

###### 2.5 全局哈希表查找

快速、慢速流程都没找到缓存就会来到这步——在系统保存的哈希表进行链式查找

```objective-c
static SyncData* id2data(id object, enum usage why)
{
    ...
    lockp->lock();
    {
        SyncData* p;
        SyncData* firstUnused = NULL;
        for (p = *listp; p != NULL; p = p->nextData) {
            if ( p->object == object ) {
                result = p;
                // atomic because may collide with concurrent RELEASE
                OSAtomicIncrement32Barrier(&result->threadCount);
                goto done;
            }
            if ( (firstUnused == NULL) && (p->threadCount == 0) )
                firstUnused = p;
        }
    
        // no SyncData currently associated with object
        if ( (why == RELEASE) || (why == CHECK) )
            goto done;
    
        // an unused one was found, use it
        if ( firstUnused != NULL ) {
            result = firstUnused;
            result->object = (objc_object *)object;
            result->threadCount = 1;
            goto done;
        }
    }
    ...
}
```

1. `lockp->lock()`并不是在底层对锁进行了封装，而是在查找过程前后进行了加锁操作
2. `for循环`遍历链表，如果有符合的就`goto done`
   - 寻找链表中未使用的`SyncData`并作标记
3. 如果是`RELEASE`或`CHECK`直接`goto done`
4. 如果第二步中有发现第一次使用的的对象就将`threadCount`标记为1且`goto done`

###### 2.6 生成新数据并写入缓存

```objective-c
static SyncData* id2data(id object, enum usage why)
{
    ...
    posix_memalign((void **)&result, alignof(SyncData), sizeof(SyncData));
    result->object = (objc_object *)object;
    result->threadCount = 1;
    new (&result->mutex) recursive_mutex_t(fork_unsafe_lock);
    result->nextData = *listp;
    *listp = result;
    
 done:
    lockp->unlock();
    if (result) {
        // Only new ACQUIRE should get here.
        // All RELEASE and CHECK and recursive ACQUIRE are 
        // handled by the per-thread caches above.
        if (why == RELEASE) {
            // Probably some thread is incorrectly exiting 
            // while the object is held by another thread.
            return nil;
        }
        if (why != ACQUIRE) _objc_fatal("id2data is buggy");
        if (result->object != object) _objc_fatal("id2data is buggy");

#if SUPPORT_DIRECT_THREAD_KEYS
        if (!fastCacheOccupied) {
            // Save in fast thread cache
            tls_set_direct(SYNC_DATA_DIRECT_KEY, result);
            tls_set_direct(SYNC_COUNT_DIRECT_KEY, (void*)1);
        } else 
#endif
        {
            // Save in thread cache
            if (!cache) cache = fetch_cache(YES);
            cache->list[cache->used].data = result;
            cache->list[cache->used].lockCount = 1;
            cache->used++;
        }
    }
    ...
}
```

1. 第三步情况均不满足（即链表不存在——对象对于全部线程来说是第一次加锁）就会创建`SyncData`并存在`result`里，方便下次进行存储

2. done分析：

   - 先将前面的lock锁解开

   - 如果是`RELEASE`类型直接返回nil

   - 对`ACQUIRE`类型和对象的断言判断

   - `!fastCacheOccupied`分支表示支持快速缓存且快速缓存被占用了，将该`SyncCacheItem`数据写入快速缓存中

   - 否则将该`SyncCacheItem`存入该线程对应的`SyncCache`中

     

     2.7 疑难解答

1. 不能使用`非OC对象`作为加锁条件——`id2data`中接收参数为id类型
2. 多次锁同一个对象会有什么后果吗——会从高速缓存中拿到data，所以只会锁一次对象
3. 都说@synchronized性能低——是因为在底层`增删改查`消耗了大量性能
4. 加锁对象不能为nil，否则加锁无效，不能保证线程安全

```objective-c
- (void)test {
    _testArray = [NSMutableArray array];
    for (int i = 0; i < 200000; i++) {
        dispatch_async(dispatch_get_global_queue(0, 0), ^{
            @synchronized (self.testArray) {
                self.testArray = [NSMutableArray array];
            }
        });
    }
}
```

上面代码一运行就会崩溃，原因是因为在某一瞬间`testArray`释放了为nil，但哈希表中存的对象也变成了nil，导致`synchronized`无效化

解决方案：

- 对`self`进行同步锁，这个似乎太臃肿了
- 使用`NSLock`

#### 3.NSLock

###### 3.1 使用

`NSLock`是对`互斥锁`的简单封装，使用如下：

```objective-c
- (void)test {
    self.testArray = [NSMutableArray array];
    NSLock *lock = [[NSLock alloc] init];
    for (int i = 0; i < 200000; i++) {
        dispatch_async(dispatch_get_global_queue(0, 0), ^{
            [lock lock];
            self.testArray = [NSMutableArray array];
            [lock unlock];
        });
    }
}

```

`NSLock`在[AFNetworking的AFURLSessionManager.m](https://github.com/AFNetworking/AFNetworking/blob/master/AFNetworking/AFURLSessionManager.m)中有使用到

想要了解一下`NSLock`的底层原理，但发现其是在未开源的`Foundation`源码下面的，但但是Swift对`Foundation`却开源了，可以在[swift-corelibs-foundation](https://github.com/apple/swift-corelibs-foundation)下载到源码来一探究竟

![img](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-03-04-092326.jpg)

从源码来看就是对互斥锁的简单封装



###### 3.2 注意事项

使用互斥锁`NSLock`异步并发调用block块，block块内部递归调用自己，问打印什么？

```objective-c
- (void)test {
    NSLock *lock = [[NSLock alloc] init];
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        static void (^block)(int);
        
        block = ^(int value) {
            NSLog(@"加锁前");
            [lock lock];
            NSLog(@"加锁后");
            if (value > 0) {
                NSLog(@"value——%d", value);
                block(value - 1);
            }
            [lock unlock];
        };
        block(10);
    });
}

```

输出结果并没有按代码表面的想法去走，而是只打印了一次value值

```shell
加锁前
加锁后
value——10
加锁前

```

**原因：** 互斥锁在递归调用时会造成堵塞，并非死锁——这里的问题是后面的代码无法执行下去

- 第一次加完锁之后还没出锁就进行递归调用
- 第二次加锁就堵塞了线程（因为不会查询缓存）

**解决方案：** 使用递归锁`NSRecursiveLock`替换`NSLock`

#### 4.NSRecursiveLock

###### 4.1 使用

`NSRecursiveLock`使用和`NSLock`类似，如下代码就能解决上个问题

```objective-c
- (void)test {
    NSRecursiveLock *lock = [[NSRecursiveLock alloc] init];
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        static void (^block)(int);
        
        block = ^(int value) {
            [lock lock];
            if (value > 0) {
                NSLog(@"value——%d", value);
                block(value - 1);
            }
            [lock unlock];
        };
        block(10);
    });
}

```

`NSRecursiveLock`在[YYKit中YYWebImageOperation.m](https://github.com/ibireme/YYKit/blob/4e1bd1cfcdb3331244b219cbd37cc9b1ccb62b7a/YYKit/Image/YYWebImageOperation.m)中有用到

###### 4.2 注意事项

递归锁在使用时需要注意死锁问题——前后代码相互等待便会产生死锁

上述代码在外层加个`for循环`，问输出结果？

```objective-c
- (void)test {
    NSRecursiveLock *lock = [[NSRecursiveLock alloc] init];
    for (int i = 0; i < 10; i++) {
        dispatch_async(dispatch_get_global_queue(0, 0), ^{
            static void (^block)(int);
            
            block = ^(int value) {
                [lock lock];
                if (value > 0) {
                    NSLog(@"value——%d", value);
                    block(value - 1);
                }
                [lock unlock];
            };
            block(10);
        });
    }
}
```

运行代码会崩溃，并会提示`野指针`错误

![img](https://user-gold-cdn.xitu.io/2020/5/23/172421e6eaef52c7?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

**原因：** for循环在block内部对同一个对象进行了多次锁操作，直到这个资源身上挂着N把锁，最后大家都无法一次性解锁——找不到解锁的出口



即 线程1中加锁1、同时线程2中加锁2-> 解锁1等待解锁2 -> 解锁2等待解锁1 -> 无法结束解锁——形成死锁

**解决：** 可以采用使用缓存的`@synchronized`，因为它对对象进行锁操作，会先从缓存查找是否有锁`syncData`存在。如果有，直接返回而不加锁，保证锁的唯一性

#### 5.dispatch_semaphore

在[GCD应用篇章](https://juejin.im/post/6844904122068500487#heading-25)已经对信号量进行过讲解

#### 6.NSCondition

`NSCondition`是一个条件锁，可能平时用的不多，但与信号量相似：线程1需要等到条件1满足才会往下走，否则就会堵塞等待，直至条件满足

同样的能在`Swift源码`中找到关于`NSCondition`部分

```swift
open class NSCondition: NSObject, NSLocking {
    internal var mutex = _MutexPointer.allocate(capacity: 1)
    internal var cond = _ConditionVariablePointer.allocate(capacity: 1)

    public override init() {
        pthread_mutex_init(mutex, nil)
        pthread_cond_init(cond, nil)
    }
    
    deinit {
        pthread_mutex_destroy(mutex)
        pthread_cond_destroy(cond)
    }
    
    open func lock() {
        pthread_mutex_lock(mutex)
    }
    
    open func unlock() {
        pthread_mutex_unlock(mutex)
    }
    
    open func wait() {
        pthread_cond_wait(cond, mutex)
    }

    open func wait(until limit: Date) -> Bool {
        guard var timeout = timeSpecFrom(date: limit) else {
            return false
        }
        return pthread_cond_timedwait(cond, mutex, &timeout) == 0
    }
    
    open func signal() {
        pthread_cond_signal(cond)
    }
    
    open func broadcast() {
        pthread_cond_broadcast(cond) // wait  signal
    }
    
    open var name: String?
}

```

从上述精简后的代码可以得出以下几点：

- `NSCondition`是对`mutex`和`cond`的一种封装（`cond`就是用于访问和操作特定类型数据的指针）
- `wait`操作会阻塞线程，使其进入休眠状态，直至超时
- `signal`操作是唤醒一个正在休眠等待的线程
- `broadcast`会唤醒所有正在等待的线程

#### 7.NSConditionLock

顾名思义，就是`NSCondition` + `Lock`

那么和`NSCondition`的区别在于哪里呢？接下来看一下`NSConditionLock`源码

```swift
open class NSConditionLock : NSObject, NSLocking {
    internal var _cond = NSCondition()
    internal var _value: Int
    internal var _thread: _swift_CFThreadRef?
    
    public convenience override init() {
        self.init(condition: 0)
    }
    
    public init(condition: Int) {
        _value = condition
    }

    open func lock() {
        let _ = lock(before: Date.distantFuture)
    }

    open func unlock() {
        _cond.lock()
        _thread = nil
        _cond.broadcast()
        _cond.unlock()
    }
    
    open var condition: Int {
        return _value
    }

    open func lock(whenCondition condition: Int) {
        let _ = lock(whenCondition: condition, before: Date.distantFuture)
    }

    open func `try`() -> Bool {
        return lock(before: Date.distantPast)
    }
    
    open func tryLock(whenCondition condition: Int) -> Bool {
        return lock(whenCondition: condition, before: Date.distantPast)
    }

    open func unlock(withCondition condition: Int) {
        _cond.lock()
        _thread = nil
        _value = condition
        _cond.broadcast()
        _cond.unlock()
    }

    open func lock(before limit: Date) -> Bool {
        _cond.lock()
        while _thread != nil {
            if !_cond.wait(until: limit) {
                _cond.unlock()
                return false
            }
        }
        _thread = pthread_self()
        _cond.unlock()
        return true
    }
    
    open func lock(whenCondition condition: Int, before limit: Date) -> Bool {
        _cond.lock()
        while _thread != nil || _value != condition {
            if !_cond.wait(until: limit) {
                _cond.unlock()
                return false
            }
        }
        _thread = pthread_self()
        _cond.unlock()
        return true
    }
    
    open var name: String?
}
```

从上述代码可以得出以下几点：

- `NSConditionLock`是`NSCondition`加线程数的封装
- `NSConditionLock`可以设置锁条件，而`NSCondition`只是无脑的通知信号

#### 8.os_unfair_lock

由于`OSSpinLock`自旋锁的bug，替代方案是内部封装了`os_unfair_lock`，而`os_unfair_lock`在加锁时会处于休眠状态，而不是自旋锁的忙等状态

#### 9.互斥锁性能对比



![img](https://user-gold-cdn.xitu.io/2020/5/24/172447ad4a4c4340?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)





## 锁

### @synchronized

一般在创建单例对象的时候使用

### atomic

修饰属性的关键字
对被修饰对象进行原子操作(不负责使用)

### OSSpinLock 自旋锁

循环等待访问，不释放当前资源(while循环)
用于轻量级数据访问,简单的int值 +1/-1操作

### NSLock

```objective-c
-(void)methodA{
    [lock lock];
    [self methodB];
    [lock unlock];
}

-(void)methodB{
    [lock lock];
    //xxxx
    [lock unlock];
}

//会导致死锁,要使用递归锁

```

### NSRescursiveLock 递归锁

```objective-c
//递归锁的特点是可以重入
-(void)methodA{
    [recursiveLock lock];
    [self methodB];
    [recursiveLock unlock];
}

-(void)methodB{
    [recursiveLock lock];
    //xxxx
    [recursiveLock unlock];
}
```

### dispatch_semaphore_t 信号量

阻塞是一个主动行为
唤醒是一个被动行为



## iOS系统为我们提供的几钟多线程技术各自的特点是怎样的

iOS系统当中主要提供3种,`GCD`、`NSOperation&NSOperationQueue`、`NSThread`,一般使用 `GCD`实现简单线程同步，包括子线程分派，实现多读单写情景，`NSOperation`方便任务状态控制，添加依赖移除依赖,`NSThread`多用于常用线程



## 总结

- `OSSpinLock`不再安全，底层用`os_unfair_lock`替代
- `atomic`只能保证setter、getter时线程安全，所以更多的使用`nonatomic`来修饰
- `读写锁`更多使用栅栏函数来实现
- `@synchronized`在底层维护了一个哈希链表进行`data`的存储，使用`recursive_mutex_t`进行加锁
- `NSLock`、`NSRecursiveLock`、`NSCondition`和`NSConditionLock`底层都是对`pthread_mutex`的封装
- `NSCondition`和`NSConditionLock`是条件锁，当满足某一个条件时才能进行操作，和信号量`dispatch_semaphore`类似
- 普通场景下涉及到线程安全，可以用`NSLock`
- 循环调用时用`NSRecursiveLock`
- 循环调用且有线程影响时，请注意死锁，如果有死锁问题请使用`@synchronized`





## Reference

[1. iOS探索 细数iOS中的那些锁](https://juejin.cn/post/6844904167010467854)