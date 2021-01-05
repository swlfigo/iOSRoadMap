# Category

可以动态地为已有类添加新行为。Apple还推荐了category的另外两个使用场景

- 可以把类的实现分开在几个不同的文件里面。这样做有几个显而易见的好处，
  - a)可以减少单个文件的体积
  -  b)可以把不同的功能组织到不同的category里 
  - c)可以由多个开发者共同完成一个类 
  - d)可以按需加载想要的category 等等。
- 声明私有方法



## Extension && Category

#### 1.Category的特点

- **运行时决议**
  - 通过 `runtime` 动态将分类的方法合并到类对象、元类对象中
  - 实例方法合并到类对象中，类方法合并到元类对象中
- 可以为系统类添加分类

#### 2.分类中可以添加哪些内容

- 实例方法
- 类方法
- 协议
- 属性



#### *Class Extension(扩展)*

- 声明私有属性
- 声明私有方法
- 声明私有成员变量
- 编译时决议，Category 运行时决议
- 不能为系统类添加扩展
- 只能以声明的形式存在，多数情况下，寄生于宿主类的.m文件中





 extension在**编译期决议**，它就是类的一部分，在编译期和头文件里的@interface以及实现文件里的@implement一起形成一个完整的类，它伴随类的产生而产生，亦随之一起消亡。extension一般用来隐藏类的私有信息，你必须有一个类的源码才能为一个类添加extension，所以你无法为系统的类比如NSString添加extension。

但是category则完全不一样，它是在运行期决议的。 就category和extension的区别来看，我们可以推导出一个明显的事实，extension可以添加实例变量，而category是无法添加实例变量的（因为在运行期，对象的内存布局已经确定，如果添加实例变量就会破坏类的内部布局，这对编译型语言来说是灾难性的）。



## Category Runtime 结构

使用 `xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc MNPerson+Test.m` 函数，生产一个cpp文件,窥探其底层结构(编译状态)

```objective-c
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

```

- **每个分类都是独立的**
- **每个分类的结构都一致**，都是`category_t`

## -category如何加载

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-01-05-033906.jpg)



```objective-c
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

```



```objective-c
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
```



画图分析

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-01-05-091909.jpg)

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-01-05-091928.jpg)

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-01-05-091943.jpg)

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-01-05-092006.jpg)

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-01-05-092019.jpg)



category被附加到类上面是在map_images的时候发生的

要注意的有两点：

1)、category的方法没有“完全替换掉”原来类已经有的方法，也就是说如果category和原来类都有methodA，那么category附加完成之后，类的方法列表里会有两个methodA

2)、category的方法被放到了新方法列表的前面，而原来类的方法被放到了新方法列表的后面，这也就是我们平常所说的category的方法会“覆盖”掉原来类的同名方法，这是因为运行时在查找方法的时候是顺着方法列表的顺序查找的，它只要一找到对应名字的方法，就会罢休，殊不知后面可能还有一样名字的方法。

###  

### Category的加载处理流程

**分类的加载处理流程主要有下面三步：**

**1.通过Runtime加载某个类的所有Category数据**
 **2.把所有Category的方法、属性、协议数据，合并到一个大数组中 后面参与编译的Category数据，会在数组的前面 **

**3.将合并后的分类数据（方法、属性、协议），插入到类原来数据的前面**



## 旁枝末叶-category和+load方法

我们知道，在类和category中都可以有+load方法，那么有两个问题：

1)、在类的+load方法调用的时候，我们可以调用category中声明的方法么？

2)、这么些个+load方法，调用顺序是咋样的呢？

答：

 1)、可以调用，因为附加category到类的工作会先于+load方法的执行 

2)、加载顺序是父类先+load，然后子类+load，然后分类+load，+load的执行顺序是先类，后category，而category的+load执行顺序是根据编译顺序决定的。

实际调用时，调用的是后添加的方法，即后添加的方法在方法列表methodLists的这个数组的顶部

后+load的类的方法，后添加到方法列表，而这时的添加方式又是插入顶部添加，即

`[methodLists insertObject:category_method atIndex:0]; `所以objc_msgSend遍历方法列表查找SEL 对应的IMP时，**会先找到分类重写的那个，调用执行。然后添加到缓存列表中，这样主类方法实现永远也不会调到。**

(后编译的Category，插入的方法在每个类大方法数组最前面)

## Reference

[1.深入理解Objective-C：Category](https://tech.meituan.com/2015/03/03/diveintocategory.html)

[2. 面试驱动技术 - Category 相关考点(Article文件夹有收藏)](https://juejin.im/post/5c753bc251882505d52fba5c)