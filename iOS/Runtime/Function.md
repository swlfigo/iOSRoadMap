# Function 函数



OC 是一门动态语言，函数调用变成了消息发送，在编译期不能知道要调用哪个函数。所以 Runtime 无非就是去解决如何在运行时期找到调用方法这样的问题

> instance -> class -> method -> SEL -> IMP -> 实现函数

根据`isa`特性可以解释消息传递与寻找方法列表原理

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-19-061439.jpg)



这就是消息传递的一个流程，首先查缓存，无缓存，查方法列表，依然没命中，再顺次查找各个父类方法列表，如果都没有名字，就转到消息转发流程

- 在缓存查找阶段是 哈希查找
- 当前类方法查找 ， 如果是已排序的列表，就采用二分查找,没排序的采用一般遍历
- 逐级父类方法查找 ，是根据 superClass 指针逐级遍历每一个父类

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2020-04-23-144612.jpg)



上述源码中，`method_array_t、property_array_t、protocol_array_t`其实都是二维数组，来到`method_array_t、property_array_t、protocol_array_t`内部看一下。这里以`method_array_t`为例，`method_array_t`本身就是一个数组，数组里面存放的是数 `method_list_t`，`method_list_t`里面最终存放的是`method_t`



```cpp
class method_array_t : 
    public list_array_tt<method_t, method_list_t> 
{
    typedef list_array_tt<method_t, method_list_t> Super;

 public:
    method_list_t **beginCategoryMethodLists() {
        return beginLists();
    }
    
    method_list_t **endCategoryMethodLists(Class cls);

    method_array_t duplicate() {
        return Super::duplicate<method_array_t>();
    }
};


class property_array_t : 
    public list_array_tt<property_t, property_list_t> 
{
    typedef list_array_tt<property_t, property_list_t> Super;

 public:
    property_array_t duplicate() {
        return Super::duplicate<property_array_t>();
    }
};


class protocol_array_t : 
    public list_array_tt<protocol_ref_t, protocol_list_t> 
{
    typedef list_array_tt<protocol_ref_t, protocol_list_t> Super;

 public:
    protocol_array_t duplicate() {
        return Super::duplicate<protocol_array_t>();
    }
};
```

`class_rw_t`里面的`methods、properties、protocols`是二维数组，是可读可写的，其中包含了类的初始内容以及分类的内容。

这里以`method_array_t`为例，图示其中的结构。

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-01-28-143707.jpg)



### class_rw_t中是如何存储方法的

### method_t

我们知道`method_array_t、property_array_t、protocol_array_t`中以`method_array_t`为例，`method_array_t`中最终存储的是`method_t`，`method_t`是对方法、函数的封装，每一个方法对象就是一个`method_t`。通过源码看一下`method_t`的结构体



```cpp
struct method_t {
    SEL name;  // 函数名
    const char *types;  // 编码（返回值类型，参数类型）
    IMP imp; // 指向函数的指针（函数地址）
};
```

method_t结构体中可以看到三个成员变量，我们依次来看三个成员变量分别代表什么。

#### SEL

SEL代表方法\函数名，一般叫做选择器，底层结构跟`char *`类似
 `typedef struct objc_selector *SEL;`，可以把SEL看做是方法名字符串。

SEL可以通过`@selector()`和`sel_registerName()`获得



```kotlin
SEL sel1 = @selector(test);
SEL sel2 = sel_registerName("test");
```

也可以通过`sel_getName()`和`NSStringFromSelector()`将SEL转成字符串



```objectivec
char *string = sel_getName(sel1);
NSString *string2 = NSStringFromSelector(sel2);
```

不同类中相同名字的方法，所对应的方法选择器是相同的。



```css
NSLog(@"%p,%p", sel1,sel2);
Runtime-test[23738:8888825] 0x1017718a3,0x1017718a3
```

SEL仅仅代表方法的名字，并且不同类中相同的方法名的SEL是全局唯一的。



#### types

`types`包含了函数返回值，参数编码的字符串。通过字符串拼接的方式将返回值和参数拼接成一个字符串，来代表函数返回值及参数。

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-01-31-135309.jpg)

#### IMP

`IMP`代表函数的具体实现，存储的内容是函数地址。也就是说当找到`imp`的时候就可以找到函数实现，进而对函数进行调用。



### 方法缓存 cache_t

回到类对象结构体，成员变量`cache`就是用来对方法进行缓存的。

```cpp
struct objc_class : objc_object {
    // Class ISA;
    Class superclass;
    cache_t cache;             // formerly cache pointer and vtable
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags

    class_rw_t *data() { 
        return bits.data();
    }
    void setData(class_rw_t *newData) {
        bits.setData(newData);
    }
}
```

**`cache_t cache;`用来缓存曾经调用过的方法，可以提高方法的查找速度。**

回顾方法调用过程：调用方法的时候，需要去方法列表里面进行遍历查找。如果方法不在列表里面，就会通过`superclass`找到父类的类对象，在去父类类对象方法列表里面遍历查找。

如果方法需要调用很多次的话，那就相当于每次调用都需要去遍历多次方法列表，为了能够快速查找方法，`apple`设计了`cache_t`来进行方法缓存。

每当调用方法的时候，会先去`cache`中查找是否有缓存的方法，如果没有缓存，在去类对象方法列表中查找，以此类推直到找到方法之后，就会将方法直接存储在`cache`中，下一次在调用这个方法的时候，就会在类对象的`cache`里面找到这个方法，直接调用了。



#### cache_t 如何进行缓存

```cpp
struct cache_t {
    struct bucket_t *_buckets; // 散列表 数组
    mask_t _mask; // 散列表的长度 -1
    mask_t _occupied; // 已经缓存的方法数量
};
```

`bucket_t`是以数组的方式存储方法列表的

```cpp
struct bucket_t {
private:
    cache_key_t _key; // SEL作为Key
    IMP _imp; // 函数的内存地址
};
```

源码中可以看出`bucket_t`中存储着`SEL`和`_imp`，通过`key->value`的形式，以`SEL`为`key`，`函数实现的内存地址 _imp`为`value`来存储方法。

通过一张图来展示一下`cache_t`的结构。

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-01-31-145738.jpg)

上述`bucket_t`列表我们称之为散列表（哈希表）
 **散列表（Hash table，也叫哈希表），是根据关键码值(Key value)而直接进行访问的数据结构。也就是说，它通过把关键码值映射到表中一个位置来访问记录，以加快查找的速度。这个映射函数叫做散列函数，存放记录的数组叫做散列表。**



#### 1.方法缓存存在什么地方？

在objc中，class存储类的实例方法（-），meta class存储类的类方法（+），class的isa指针指向meta class。

在类的定义里就有cache字段，类的所有缓存都存在metaclass上，所以每个类都只有一份方法缓存，而**不是每一个类的object都保存一份**

#### 2.父类方法的缓存只存在父类么，还是子类也会缓存父类的方法？

即便是从父类取到的方法，**也会存在类本身的方法缓存里**。而当用一个父类对象去调用那个方法的时候，也会在父类的metaclass里缓存一份。

#### 3.为什么 类的方法列表 不直接做成散列表呢，做成list，还要单独缓存，多费事？

- 散列表是没有顺序的，Objective-C的方法列表是一个list，是有顺序的；Objective-C在查找方法的时候会顺着list依次寻找，并且category的方法在原始方法list的前面，需要先被找到，如果直接用hash存方法，方法的顺序就没法保证。
- list的方法还保存了除了selector和imp之外其他很多属性
- 散列表是有空槽的，会浪费空间



## Reference

[1.iOS底层原理总结 - 探寻Runtime本质（二）](https://www.jianshu.com/p/27ee04f3ed7b)


