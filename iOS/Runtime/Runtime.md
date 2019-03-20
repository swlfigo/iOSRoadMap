# Runtime机制

## List

<a href="#Runtime Foundation">1. Runtime基础</a>

<a href="#Runtime Key Points">2.Runtime一些考点与知识点</a>

<a href="#Runtime Reference">3.Runtime Reference</a>



<a id="Runtime Foundation">

## Runtime基础

Runtime内容流程

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-19-022544.jpg)

## 1.对象和类数据结构

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-19-023218.jpg)

一个类的的结构体组成结构如上

- superClass 指向当前类的父类
- cache_t 提供消息传递过程当中的缓存方法查找 ， 实质上是装满了 bucket_t 的一个 **hash 表**。因为散列表检索起来更快，
- class_data_bits_t 类的基础信息，包含了类的方法列表，协议列表等。

#### 1.1 class_data_bits_t 结构体

`class_data_bits_t`结构体中只有一个64位的指针bits，它相当于`class_rw_t `指针加上 rr/alloc 等标志位。其中`class_rw_t`指针存在于4~47位（从1开始计）

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-19-023930.png)

class_rw_t结构体的定义如下

```objective-c
struct class_rw_t {
    uint32_t flags;
    uint32_t version;

    const class_ro_t *ro;
		//这个是单个类(基本原类)的结构体
  //里面与 class_rw_t结构一样
  //也是包括了方法，属性与协议
  
    method_array_t methods;
  //重点看这里
  //方法数组.
  //其实是该类与该类的分类所有方法的集合
  //下 属性与协议数组相同 意思
    property_array_t properties;
    protocol_array_t protocols;

    Class firstSubclass;
}；
```



#### 1.2 class_ro_t 结构体

class_ro_t与class_rw_t的最大区别在于一个是只读的，一个是可读写的，实质上ro就是readonly的简写，rw是readwrite的简写。

```objective-c
struct class_ro_t {
    const char * name;
    method_list_t * baseMethodList;
    protocol_list_t * baseProtocols;
    const ivar_list_t * ivars;
};
```

`class_ro_t`在**内存中是不可变**的。**[所以说扩展编译器已经决定了内存结构不能添加属性，只能通过关联添加]**在运行期间，动态给类添加方法，实质上是更新class_rw_t的methods列表。



## 2.类对象与元类对象

#### 2.1 metaclass & class

在objc中，class存储类的实例方法（-），meta class存储类的类方法（+），class的isa指针指向meta class。

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-19-023919.jpg)



每个类**仅有一个类对象**，而每个类对象**仅有一个与之相关的元类**。当你发出一个类似 `[NSObject alloc] `的消息时，你事实上是把这个消息发给了一个类对象 (Class Object) ，这个类对象必须是一个元类的实例，而这个元类同时也是一个根元类 (root meta class) 的实例。所有的元类最终都指向根元类为其超类。所有的元类的方法列表都有能够响应消息的类方法。所以当 [NSObject alloc] 这条消息发给类对象的时候，objc_msgSend() 会去它的元类里面去查找能够响应消息的方法，如果找到了，然后对这个类对象执行方法调用。

看一个例子:

```objective-c
@interface CustomObject : NSObject
-(NSString *)returnMeHelloWorld;
@end
@implementation CustomObject

-(NSString *)returnMeHelloWorld{
    return @"hello world";
}
@end
```

先只看调用这一行:

```objective-c
 NSString * helloWorld =  [obj returnMeHelloWorld];
```

1. 编译成如下`id objc_msgSend(self,@selector(returnMeHelloWorld:));`
2. 在self中沿着`isa`找到`CustomObject`的类对象
3. 类对象查找自己的方法list，找到对应的方法执行体Method[还要缓存一下]
4. 把参数传递给IMP实际指向的执行代码
5. 代码执行返回结果给helloWorld

## 3.消息传递

OC 是一门动态语言，函数调用变成了消息发送，在编译期不能知道要调用哪个函数。所以 Runtime 无非就是去解决如何在运行时期找到调用方法这样的问题

> instance -> class -> method -> SEL -> IMP -> 实现函数

根据`isa`特性可以解释消息传递与寻找方法列表原理

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-19-061439.jpg)



这就是消息传递的一个流程，首先查缓存，无缓存，查方法列表，依然没命中，再顺次查找各个父类方法列表，如果都没有名字，就转到消息转发流程

- 在缓存查找阶段是 哈希查找
- 当前类方法查找 ， 如果是已排序的列表，就采用二分查找,没排序的采用一般遍历
- 逐级父类方法查找 ，是根据 superClass 指针逐级遍历每一个父类



引申一个考点

**Q:OC中向一个nil对象发送消息将会发生什么?为什么?**

**A:** nil的定义是null pointer to object-c object，指的是一个OC对象指针为空，本质就是(id)0，是OC对象的字面0值.OC中给空指针发消息不会崩溃的语言特性，**原因是OC的函数调用都是通过objc_msgSend进行消息发送来实现的**，相对于C和C++来说，对于空指针的操作会引起Crash的问题，**而objc_msgSend会通过判断self来决定是否发送消息，如果self为nil，那么selector也会为空，直接返回**，所以不会出现问题。

objc在向一个对象发送消息时，runtime库会根据对象的isa指针找到该对象实际所属的类，然后在该类中的方法列表以及其父类方法列表中寻找方法运行，然后在发送消息的时候，objc_msgSend方法不会返回值，所谓的返回内容都是具体调用时执行的。那么，回到本题，如果向一个nil对象发送消息，首先在寻找对象的isa指针时就是0地址返回了，所以不会出现任何错误。



### 3.1 消息转发

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-19-061727.jpg)



<a id="Runtime Key Points">

## Runtime更多知识点

#### 1.方法缓存存在什么地方？

在类的定义里就有cache字段，没错，类的所有缓存都存在metaclass上，所以每个类都只有一份方法缓存，而**不是每一个类的object都保存一份**

#### 2.父类方法的缓存只存在父类么，还是子类也会缓存父类的方法？

即便是从父类取到的方法，**也会存在类本身的方法缓存里**。而当用一个父类对象去调用那个方法的时候，也会在父类的metaclass里缓存一份。

#### 3.为什么 类的方法列表 不直接做成散列表呢，做成list，还要单独缓存，多费事？

- 散列表是没有顺序的，Objective-C的方法列表是一个list，是有顺序的；Objective-C在查找方法的时候会顺着list依次寻找，并且category的方法在原始方法list的前面，需要先被找到，如果直接用hash存方法，方法的顺序就没法保证。
- list的方法还保存了除了selector和imp之外其他很多属性
- 散列表是有空槽的，会浪费空间



<a id="Runtime Reference">

## Reference

[1.Objective-C Runtime机制简析(Article文件夹有收藏)](https://www.jianshu.com/p/0a4e5b944d7d)

[2.Objective-C Runtime](http://yulingtianxia.com/blog/2014/11/05/objective-c-runtime/)

[3.深入理解 Objective-C：方法缓存](https://tech.meituan.com/2015/08/12/deep-understanding-object-c-of-method-caching.html)

