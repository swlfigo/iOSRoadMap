> 面试驱动技术合集（初中级 iOS 开发），关注仓库，及时获取更新 [Interview-series](https://link.juejin.im?target=https%3A%2F%2Fgithub.com%2FminiLV%2FInterview-series)

[原文: 面试驱动技术 - Category 相关考点](https://juejin.im/post/5c753bc251882505d52fba5c)

_Category 相关的问题一般初中级问的比较多，一般最深的就问到关联对象，本文把比较常见的 Category 的问题都罗列解决了一下，如果还有其他常见的 Category 的试题欢迎补充~_

## I. Category

### Category 相关面试题

* Category 实现原理？
* 实际开发中，你用 Category 做了哪些事？
* Category 能否添加成员变量，如果可以，如何添加？
* load 、initialize 方法的区别是什么，他们在 category 中的调用顺序？以及出现继承时他们之间的调用过程？
* Category 和 Class Extension 的区别是什么？
* 为什么分类会 “覆盖” 宿主类的方法？

  


#### 1.Category 的特点

*   运行时决议
    *   通过 `runtime` 动态将分类的方法合并到类对象、元类对象中
    *   实例方法合并到类对象中，类方法合并到元类对象中
*   可以为系统类添加分类

#### 2\. 分类中可以添加哪些内容

*   实例方法
*   类方法
*   协议
*   属性

### 分类中原理解析

使用 `xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc MNPerson+Test.m` 函数，生产一个 cpp 文件, 窥探其底层结构 (编译状态)

```
struct _category_t {
    //宿主类名称 - 这里的MNPerson
    const char *name;

    //宿主类对象,里面有isa
    struct _class_t *cls;

    //实例方法列表
    const struct _method_list_t *instance_methods;

    //类方法列表
    const struct _method_list_t *class_methods;

    //协议列表
    const struct _protocol_list_t *protocols;

    //属性列表
    const struct _prop_list_t *properties;
};

//_class_t 结构
struct _class_t {
	struct _class_t *isa;
	struct _class_t *superclass;
	void *cache;
	void *vtable;
	struct _class_ro_t *ro;
};
复制代码
```

*   每个分类都是独立的
*   每个分类的结构都一致，都是`category_t`

#### 函数转换

```
@implementation MNPerson (Test)

- (void)test{
    NSLog(@"test - rua~");
}

@end
复制代码
```

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-14-040902.jpg)



```
static void 
attachCategories(Class cls, category_list *cats, bool flush_caches)
{
    if (!cats) return;
    if (PrintReplacedMethods) printReplacements(cls, cats);

    bool isMeta = cls->isMetaClass();

    // fixme rearrange to remove these intermediate allocations

    /* 二维数组( **mlists => 两颗星星，一个)
     [
        [method_t,],
        [method_t,method_t],
        [method_t,method_t,method_t],
     ]

     */
    method_list_t **mlists = (method_list_t **)
        malloc(cats->count * sizeof(*mlists));
    property_list_t **proplists = (property_list_t **)
        malloc(cats->count * sizeof(*proplists));
    protocol_list_t **protolists = (protocol_list_t **)
        malloc(cats->count * sizeof(*protolists));

    // Count backwards through cats to get newest categories first
    int mcount = 0;
    int propcount = 0;
    int protocount = 0;
    int i = cats->count;//宿主类，分类的总数
    bool fromBundle = NO;
    while (i--) {//倒序遍历，最先访问最后编译的分类

        // 获取某一个分类
        auto& entry = cats->list[i];

        // 分类的方法列表
        method_list_t *mlist = entry.cat->methodsForMeta(isMeta);
        if (mlist) {
            //最后编译的分类，最先添加到分类数组中
            mlists[mcount++] = mlist;
            fromBundle |= entry.hi->isBundle();
        }

        property_list_t *proplist = 
            entry.cat->propertiesForMeta(isMeta, entry.hi);
        if (proplist) {
            proplists[propcount++] = proplist;
        }

        protocol_list_t *protolist = entry.cat->protocols;
        if (protolist) {
            protolists[protocount++] = protolist;
        }
    }

    auto rw = cls->data();

    prepareMethodLists(cls, mlists, mcount, NO, fromBundle);

    // 核心：将所有分类的对象方法，附加到类对象的方法列表中
    rw->methods.attachLists(mlists, mcount);
    free(mlists);
    if (flush_caches  &&  mcount > 0) flushCaches(cls);

    rw->properties.attachLists(proplists, propcount);
    free(proplists);

    rw->protocols.attachLists(protolists, protocount);
    free(protolists);
}
复制代码
```

```
void attachLists(List* const * addedLists, uint32_t addedCount) {
    if (addedCount == 0) return;

    if (hasArray()) {
        // many lists -> many lists
        uint32_t oldCount = array()->count;
        uint32_t newCount = oldCount + addedCount;

        //realloc - 重新分配内存 - 扩容了
        setArray((array_t *)realloc(array(), array_t::byteSize(newCount)));
        array()->count = newCount;

        //memmove,内存挪动
        //array()->lists 原来的方法列表
        memmove(array()->lists + addedCount,
                array()->lists,
                oldCount * sizeof(array()->lists[0]));

        //memcpy - 将分类的方法列表 copy 到原来的方法列表中
        memcpy(array()->lists,
               addedLists,
               addedCount * sizeof(array()->lists[0]));
    }
    ...
}
复制代码
```

画图分析就是

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-14-041106.jpg)

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-14-041125.jpg)

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-14-042435.jpg)

![](https://sylarimage.oss-cn-shenzhen.aliyuncs.com/20190314121618.png)

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-14-042114.jpg)

#### 3\. 实际开发中，你用 Category 做了哪些事？

* 声明私有方法

* 分解体积庞大的类文件

* 把`Framework`的私有方法公开

  

#### 4.Category 实现原理？

*   Category 编译之后，底层结构是 category_t，里面存储着分类的各种信息，包括 对象方法、类方法、属性、协议信息
*   分类的在编译后，方法并不会直接添加到类信息中，而是要在程序运行的时候，通过 `runtime`, 讲 Category 的数据，

#### 5\. 为什么分类会 “覆盖” 宿主类的方法？

*   其实不是真正的 “覆盖”，宿主类的同名方法还是存在
*   分类将附加到类对象的方法列表中，整合的时候，分类的方法优先放到前面
*   OC 的函数调用底层走的是 msg_send() 函数，它做的是方法查找，因为分类的方法优先放在前面，所以通过选择器查找到分类的方法之后直接调用，宿主类的方法看上去就像被 “覆盖” 而没有生效

#### 6.Category 和 Class Extension 的区别是什么？

#### _Class Extension(扩展)_

* 声明私有属性
* 声明私有方法
* 声明私有成员变量
* 编译时决议，Category 运行时决议
* 不能为系统类添加扩展
* 只能以声明的形式存在，多数情况下，寄生于宿主类的. m 文件中

  


## II. load 、initialize

### load 实现原理

> *   类第一次加载进内存的时候，会调用 `+ load` 方法，无需导入，无需使用
> *   每个类、分类的 `+ load` 在程序运行过程中只会执行一次
> *   `+ load` 走的不是消息发送的 `objc_msgSend` 调用，而是找到 `+ load` 函数的地址，直接调用

```
void call_load_methods(void)
{
    static bool loading = NO;
    bool more_categories;

    loadMethodLock.assertLocked();

    // Re-entrant calls do nothing; the outermost call will finish the job.
    if (loading) return;
    loading = YES;

    void *pool = objc_autoreleasePoolPush();

    do {
        // 1\. Repeatedly call class +loads until there aren’t any more
        while (loadable_classes_used > 0) {
            //先加载宿主类的load方法(按照编译顺序，调用load方法)
            call_class_loads();
        }

        // 2\. Call category +loads ONCE
        more_categories = call_category_loads();

        // 3\. Run more +loads if there are classes OR more untried categories
    } while (loadable_classes_used > 0  ||  more_categories);

    objc_autoreleasePoolPop(pool);

    loading = NO;
}
复制代码
```

```
static void schedule_class_load(Class cls)
{
    if (!cls) return;
    assert(cls->isRealized());  // _read_images should realize

    if (cls->data()->flags & RW_LOADED) return;

    // Ensure superclass-first ordering
    // 递归调用，先将父类添加到load方法列表中，再将自己加进去
    schedule_class_load(cls->superclass);

    add_class_to_loadable_list(cls);
    cls->setInfo(RW_LOADED); 
}
复制代码
```

* * *

#### 调用顺序

1.  先调用宿主类的`+ load` 函数
    *   按照编译先后顺序调用（先编译，先调用）
    *   调用子类的 + load 之前会先调用父类的 + load
2.  再调用分类的的`+ load` 函数
    *   按照编译先后顺序调用（先编译，先调用）

实验证明：宿主类先调用，分类再调用

```
2019-02-27 17:28:00.519862+0800 load-Initialize-Demo[91107:2281575] MNPerson + load
2019-02-27 17:28:00.520032+0800 load-Initialize-Demo[91107:2281575] MNPerson (Play) + load
2019-02-27 17:28:00.520047+0800 load-Initialize-Demo[91107:2281575] MNPerson (Eat) + load
复制代码
```

![image-20190313220920309](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-13-142313.png)

* * *

```
2019-02-27 17:39:10.354050+0800 load-Initialize-Demo[91308:2303030] MNDog + load (宿主类1)
2019-02-27 17:39:10.354237+0800 load-Initialize-Demo[91308:2303030] MNPerson + load (宿主类2)
2019-02-27 17:39:10.354252+0800 load-Initialize-Demo[91308:2303030] MNDog (Rua) + load (分类1)
2019-02-27 17:39:10.354263+0800 load-Initialize-Demo[91308:2303030] MNPerson (Play) + load(分类2)
2019-02-27 17:39:10.354274+0800 load-Initialize-Demo[91308:2303030] MNPerson (Eat) + load(分类3)
2019-02-27 17:39:10.354285+0800 load-Initialize-Demo[91308:2303030] MNDog (Run) + load(分类4)
复制代码
```

#### Initialize 实现原理

> *   类第一次接收到消息的时候，会调用该方法，需导入，并使用
> *   `+ Initialize` 走的是消息发送的 `objc_msgSend` 调用

#### Initialize 题目出现

```
/*父类*/
@interface MNPerson : NSObject

@end

@implementation MNPerson

+ (void)initialize{
    NSLog(@"MNPerson + initialize");
}

@end

/*子类1*/
@interface MNTeacher : MNPerson

@end

@implementation MNTeacher

@end

/*子类2*/
@interface MNStudent : MNPerson

@end

@implementation MNStudent

@end

---------------------------------------------
问题出现:以下会输出什么结果
int main(int argc, const char * argv[]) {
    @autoreleasepool {

        [MNTeacher alloc];
        [MNStudent alloc];
    }
    return 0;
}

复制代码
```

* * *

结果如下：

```
2019-02-27 17:57:33.305655+0800 load-Initialize-Demo[91661:2331296] MNPerson + initialize
2019-02-27 17:57:33.305950+0800 load-Initialize-Demo[91661:2331296] MNPerson + initialize
2019-02-27 17:57:33.306476+0800 load-Initialize-Demo[91661:2331296] MNPerson + initialize
复制代码
```

**exo me? 为啥打印三次呢**



原理分析：

1.  `initialize` 在类第一次接收消息的时候会调用，OC 里面的 `[ xxx ]` 调用都可以看成 `objc_msgSend`, 所以这时候，`[MNTeacher alloc]` 其实内部会调用 `[MNTeacher initialize]`
2.  `initialize` 调用的时候，要先实现自己父类的 `initialize` 方法，第一次调用的时候，`MNPerson` 没被使用过，所以未被初始化，要先调用一下父类的 `[MNPerson initialize]`, 输出第一个`MNPerson + initialize`
3.  `MNPerson` 调用了 `initialize` 之后，轮到`MNTeacher` 类自己了，由于他内部没有实现 `initialize`方法，所以调用父类的`initialize`, 输出第二个`MNPerson + initialize`
4.  然后轮到`[MNStudent alloc]`，内部也是调用 `[MNStudent initialize]`, 然后判断得知 父类`MNPerson`类调用过`initialize`了，因此调用自身的就够了，由于他和`MNTeacher` 一样，也没实现`initialize` 方法，所以同理调用父类的`[MNPerson initialize]`, 输出第 3 个`MNPerson + initialize`

* * *

### initialize 与 load 的区别

*   load 是类第一次加载的时候调用，initialize 是类第一次接收到消息的时候调用，每个类只会 initialize 一次（父类的 initialize 方法可能被调用多次）
*   load 和 initialize，加载 or 调用的时候，都会先调用父类对应的 `load` or `initialize` 方法，再调用自己本身的;
*   load 和 initialize 都是系统自动调用的话，都只会调用一次
*   调用方式也不一样，load 是根据函数地址直接调用，initialize 是通过`objc_msgSend`
*   调用时刻，load 是 runtime 加载类、分类的时候调用（只会调用一次）
*   调用顺序:
    *   load:
        *   先调用类的 load
            *   先编译的类，优先调用 load
            *   调用子类的 load 之前，会先调用父类的 load
        *   在调用分类的 load
    *   initialize：
        *   先初始化父列
        *   再初始化子类（可能最终调用的是父类的初始化方法）

```
/*父类*/
@interface MNPerson : NSObject

@end

@implementation MNPerson

+ (void)initialize{
    NSLog(@"MNPerson + initialize");
}

+ (void)load{
    NSLog(@"MNPerson + load");
}

/*子类1*/
@interface MNTeacher : MNPerson

@end

@implementation MNTeacher

+ (void)load{
    NSLog(@"MNTeacher + load");
}

/*子类2*/
@interface MNStudent : MNPerson

@end

@implementation MNStudent

+ (void)load{
    NSLog(@"MNStudent + load");
}

------------------------------------
问题出现:以下会输出什么结果?

int main(int argc, const char * argv[]) {
    @autoreleasepool {

        [MNTeacher load];
    }
    return 0;
}

复制代码
```

#### 答案出现！！！

```
2019-02-27 18:17:12.034392+0800 load-Initialize-Demo[92064:2370496] MNPerson + load
2019-02-27 18:17:12.034555+0800 load-Initialize-Demo[92064:2370496] MNStudent + load
2019-02-27 18:17:12.034569+0800 load-Initialize-Demo[92064:2370496] MNTeacher + load
2019-02-27 18:17:12.034627+0800 load-Initialize-Demo[92064:2370496] MNPerson + initialize
2019-02-27 18:17:12.034645+0800 load-Initialize-Demo[92064:2370496] MNPerson + initialize
2019-02-27 18:17:12.034658+0800 load-Initialize-Demo[92064:2370496] MNTeacher + load
复制代码
```

exo me again！怎么这么多！连 load 也有了？



解释：

1.  前三个 load 不多 bb 了吧，程序一运行，runtime 直接将全部的类加载到内存中，肯定最先输出；
2.  第一个 `MNPerson + initialize`，因为是`MNTeacher`的调用，所以会先让父类`MNPerson` 调用一次`initialize`，输出第一个 `MNPerson + initialize`
3.  第二个 `MNPerson + initialize`, `MNTeacher` 自身调用，由于他自己没有实现 `initialize`, 调用父类的`initialize`， 输出第二个 `MNPerson + initialize`
4.  最后一个`MNTeacher + load`可能其实有点奇怪，不是说 `load`只会加载一次吗，而且他还不走 `objc_msgSend` 吗，怎么还能调用这个方法？
    *   因为！当类第一次加载进内存的时候，调用的 `load` 方法是系统调的，这时候不走 `objc_msgSend`
    *   但是，你现在是`[MNTeacher load]`啊，这个就是 objc_msgSend(MNTeacher,@selector(MNTeacher))，这就跑到`MNTeacher + load`里了！
    *   只是一般没人手动调用`load` 函数，但是，还是可以调用的！

## III. 关联对象 AssociatedObject

### Category 能否添加成员变量，如果可以，如何添加？

> 这道题实际上考的就是关联对象

如果是普通类声明生命属性的话

```
@interface MNPerson : NSObject

@property (nonatomic, copy)NSString *property;

@end
复制代码
```

上述代码系统内部会自动三件事：

1.  帮我们生成一个生成变量_property
2.  生成一个 `get` 方法 `- (NSString *)property`
3.  生成一个 `set` 方法 `- (void)setProperty:(NSString *)property`

```
@implementation MNPerson{
    NSString *_property;
}

- (void)setProperty:(NSString *)property{
    _property = property;
}

- (NSString *)property{
    return _property;
}

@end
复制代码
```

分类也是可以添加属性的 - 类结构里面，有个`properties` 列表，里面就是 存放属性的;

分类里面，生成属性，只会生成方法的声明，不会生成成员变量 && 方法实现！

![image-20190313221010091](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-13-142324.png)

> 人工智障翻译：实例变量不能放在分类中

所以：

**不能直接给 category 添加成员变量，但是可以间接实现分类有成员变量的效果 (效果上感觉像成员变量)**

```
@interface MNPerson (Test)

@property (nonatomic, assign) NSInteger age;

@end

@implementation MNPerson (Test)

@end
复制代码
```



![image-20190313221032988](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-13-142331.png)

`person.age = 10`等价于 `[person setAge:10]`，所以证明了，给分类声明属性之后，并没有添加其对应的实现！

### 关联对象

objc_setAssociatedObject Api

```
objc_setAssociatedObject(    <#id  _Nonnull object#>, (对象)
                             <#const void * _Nonnull key#>,(key)
                             <#id  _Nullable value#>,(关联的值)
                             <#objc_AssociationPolicy policy#>)(关联策略)
复制代码
```

关联策略，等价于属性声明

```
typedef OBJC_ENUM(uintptr_t, objc_AssociationPolicy) {
    OBJC_ASSOCIATION_ASSIGN = 0,          
    OBJC_ASSOCIATION_RETAIN_NONATOMIC = 1, 
    OBJC_ASSOCIATION_COPY_NONATOMIC = 3,  
    OBJC_ASSOCIATION_RETAIN = 01401,      
    OBJC_ASSOCIATION_COPY = 01403         
};
复制代码
```

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-14-042550.jpg)

比如这里的 age 属性，默认声明是`@property (nonatomic, assign) NSInteger age;`，就是 assign，所以这里选择`OBJC_ASSOCIATION_ASSIGN`

取值

```
objc_getAssociatedObject(<#id  _Nonnull object#>, <#const void * _Nonnull key#>)
复制代码
```

### 面试题 - 以下代码输出的结果是啥

```
int main(int argc, const char * argv[]) {
    @autoreleasepool {

        MNPerson *person = [[MNPerson alloc]init];

        {
            MNPerson *test = [[MNPerson alloc]init];
            objc_setAssociatedObject(person,
                                     @"test",
                                     test,
                                     OBJC_ASSOCIATION_ASSIGN);
        }

        NSLog(@"%@",objc_getAssociatedObject(person, @"test"));
    }
    return 0;
}

复制代码
```

![image-20190313221130229](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-13-142345.png)

> 原因，关联的对象是 person，关联的 value 是 test，test 变量 出了他们的`{}` 作用域之后，就会销毁; 此时通过 key 找到 对应的对象，访问对象内部的 value，因为 test 变量已经销毁了，所以程序崩溃了，这也说明了 => **内部 test 对 value 是强引用！**

### 关联对象的本质

> 在分类中，因为类的实例变量的布局已经固定，使用 @property 已经无法向固定的布局中添加新的实例变量（这样做可能会覆盖子类的实例变量），所以我们需要使用关联对象以及两个方法来模拟构成属性的三个要素。

引用自 [关联对象 AssociatedObject 完全解析](https://link.juejin.im?target=https%3A%2F%2Fdraveness.me%2Fao)

* * *

### 关联对象的原理

实现关联对象技术的核心对象有

*   AssociationsManager
*   AssociationsHashMap
*   ObjectAssociationMap
*   ObjcAssociation

```
class AssociationsManager {
    static spinlock_t _lock;//自旋锁，保证线程安全
    static AssociationsHashMap *_map;
}
复制代码
```

```
class AssociationsHashMap : public unordered_map<disguised_ptr_t, ObjectAssociationMap> 
复制代码
```

```
class ObjectAssociationMap : public std::map<void *, ObjcAssociation>
复制代码
```

```
class ObjcAssociation {
    uintptr_t _policy;
    id _value;
}
复制代码
```

以关联对象代码为例:

```
  objc_setAssociatedObject(obj, @selector(key), @"hello world", OBJC_ASSOCIATION_COPY_NONATOMIC);
复制代码
```

![image-20190313221149179](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-13-142352.jpg)

*   关联对象并不是存储在被关联对象本身的内存中的
*   关联对象，存储在全局的一个统一的`AssociationsManager`中
*   关联对象其实就是 `ObjcAssociation` 对象, 关联的 `value` 就放在 `ObjcAssociation` 内
*   关联对象由 `AssociationsManager` 管理并在 `AssociationsHashMap` 存储
*   对象的指针以及其对应 `ObjectAssociationMap` 以键值对的形式存储在 `AssociationsHashMap` 中
*   `ObjectAssociationMap` 则是用于存储关联对象的数据结构
*   每一个对象都有一个标记位 `has_assoc` 指示对象是否含有关联对象
*   存储在全局的一个统一的`AssociationsManager` 内部有一持有一个`_lock`，他其实是一个 spinlock_t(自旋锁), 用来保证`AssociationsHashMap`操作的时候，是线程安全的

`Category` 相关的问题一般初中级问的比较多，一般最深的就问到`关联对象`，上面的问题以及解答已经把比较常见的 `Category` 的问题都罗列解决了一下，如果还有其他常见的 `Category` 的试题欢迎补充~



* * *



参考资料:

[关联对象 AssociatedObject 完全解析](https://link.juejin.im?target=https%3A%2F%2Fdraveness.me%2Fao)

[associated-objects](https://link.juejin.im?target=https%3A%2F%2Fnshipster.com%2Fassociated-objects%2F)