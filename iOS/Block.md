#  21 Block

## 什么是Block与Block构造
* `Block` 是将函数及其执行上下文封装起来的`对象`
* `Block`的调用即是函数的调用

如：
```
int multiplier = 6
int(^Block)(int) = ^int(int num){
    return num * multiplier;
}
```

使用 `clang` 重写 

```
//会自动生成一个与定义Block具有相同参数与返回值的构造函数
int(*Block)(int) = ((int(*)(int))&xxx_block_impl_0[Block构造函数](函数指针,block描述,multiplier[block外传入block使用的参数])
```

一个`OC Block` `Clang` 重写之后，编程了一个对象的构造方法，传进去的是函数指针(`Block`的实现,与外部变量,Block描述)

`Block结构体`

```
//如上的构造函数
struct xxx_block_impl_0{
    struct __block_impl impl;
    struct __xxx_block_desc_0 *Desc;    //Block描述
    xxx_block_impl_0(void *fp,struct ....[不详细写，其实就是将上面构造函数传进去的参数传这里]){
    impl.isa = &_NSConcreteStackBlock;
    ...
    }
    //其实是C++ 里面对结构体生成的声明
}
``` 

这个`Block`结构体里面有个方法，可以理解成`OC`里面的便利构造器，上面通过`Clang`重写了`Block`的生成，其实就是调用这个结构体里面的构造函数，将参数传进去赋值给结构体里面的变量

## 截获变量

数据类型:
* 局部变量
    * 基本数据类型
    * 对象类型
* 静态局部变量
* 全局静态变量
* 静态全局变量

具体不同参考:[staic](../knowledge/staticCompare.md)

如何截取:

* 对于**基本数据**类型的**局部变量**截获其值
* 对于**对象**类型的局部变量**连同所有权修饰符**一齐截获(可以解释循环引用)
* 以**指针形式**截获局部静态变量
* **不截获**全局变量，静态全局变量


```
//全局变量
int global_var = 4;
//静态全局变量
static int static_global_var = 5;

-(void)method{
    //基本数据类型的局部变量
    int var = 1;
    //对象类型的局部变量
    __unsafe_unretained id unsafe_obj = nil;
    __strong id strong_obj = nil;
    
    //局部静态变量
    static int static_var = 3;
}
```

## __block 修饰符

**一般情况下**，对被截获变量进行**赋值**操作需要添加 `__block` 修饰符(**注意是赋值!!**, 赋值≠使用)

```
NSMutableArray *array = [NSMutableArray array];
void(^Block)(void) = ^{
    [array addObject:@123];
}

//不需要添加 __block,因为是使用
```

需要__block修饰符:
* 局部变量基本数据类型
* 局部变量对象类型

不需要__block修饰符:
* 静态局部变量
* 全局变量
* 静态全局变量

### __block原理
__block修饰的变量成了对象

![](http://img.isylar.com/media/15485830067935.jpg)

![](http://img.isylar.com/media/15485830763121.jpg)

**栈上**的 __block 的 `__forwading` 指针指向自己

## Block 的内存管理

impl.isa = &_NSConcreteStackBlock; (Block构造函数默认创建Block为 StackBlock,如上源码)

* _NSConcreteGlobalBlock (全局Block)
* _NSConcreteStackBlock (栈Block)
* _NSConcreateMallocBlock (堆Block)

 
 ![](http://img.isylar.com/media/15485833872960.jpg)


![](http://img.isylar.com/media/15485833963674.jpg)

![](http://img.isylar.com/media/15485836513812.jpg)


## __forwarding存在意义
不论在任何内存位置,都可以顺利访问同一个__block变量.
