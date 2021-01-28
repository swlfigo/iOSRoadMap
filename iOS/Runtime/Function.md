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