# Category

可以动态地为已有类添加新行为。Apple还推荐了category的另外两个使用场景

- 可以把类的实现分开在几个不同的文件里面。这样做有几个显而易见的好处，
  - a)可以减少单个文件的体积
  -  b)可以把不同的功能组织到不同的category里 
  - c)可以由多个开发者共同完成一个类 
  - d)可以按需加载想要的category 等等。
- 声明私有方法



## Extension && Category

 extension在编译期决议，它就是类的一部分，在编译期和头文件里的@interface以及实现文件里的@implement一起形成一个完整的类，它伴随类的产生而产生，亦随之一起消亡。extension一般用来隐藏类的私有信息，你必须有一个类的源码才能为一个类添加extension，所以你无法为系统的类比如NSString添加extension。

但是category则完全不一样，它是在运行期决议的。 就category和extension的区别来看，我们可以推导出一个明显的事实，extension可以添加实例变量，而category是无法添加实例变量的（因为在运行期，对象的内存布局已经确定，如果添加实例变量就会破坏类的内部布局，这对编译型语言来说是灾难性的）。



## Category Runtime 结构



所有的OC类和对象，在runtime层都是用struct表示的，category也不例外，在runtime层，category用结构体category_t（在objc-runtime-new.h中可以找到此定义），它包含了：

- 1)、类的名字（name）
- 2)、类（cls）
- 3)、category中所有给类添加的实例方法的列表（instanceMethods）
- 4)、category中所有添加的类方法的列表（classMethods）
- 5)、category实现的所有协议的列表（protocols）
- 6)、category中添加的所有属性（instanceProperties）

```c
typedef struct category_t {
    const char *name;
    classref_t cls;
    struct method_list_t *instanceMethods;
    struct method_list_t *classMethods;
    struct protocol_list_t *protocols;
    struct property_list_t *instanceProperties;
} category_t;
```

从category的定义也可以看出category的可为（可以添加实例方法，类方法，甚至可以实现协议，添加属性）和不可为（无法添加实例变量）。

## -category如何加载

对于OC运行时，入口方法如下（在objc-os.mm文件中）：

```c
void _objc_init(void)
{
    static bool initialized = false;
    if (initialized) return;
    initialized = true;
   
    // fixme defer initialization until an objc-using image is found?
    environ_init();
    tls_init();
    lock_init();
    exception_init();
       
    // Register for unmap first, in case some +load unmaps something
    _dyld_register_func_for_remove_image(&unmap_image);
    dyld_register_image_state_change_handler(dyld_image_state_bound,
                                             1/*batch*/, &map_images);
    dyld_register_image_state_change_handler(dyld_image_state_dependents_initialized, 0/*not batch*/, &load_images);
}
```

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