#   Block

## 什么是Block与Block构造

> In programming languages, a closure is a function or reference to a function together with a referencing environment—a table storing a reference to each of the non-local variables (also called free variables or upvalues) of that function.

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

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-20-051401.jpg)

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-20-051409.jpg)

**栈上**的 \__block 的  ` __forwading` 指针指向自己

## Block 的内存管理

* _NSConcreteGlobalBlock (全局的静态 block，不会访问任何外部变量)
* _NSConcreteStackBlock ( 保存在栈中的 block，当函数返回时会被销毁)
* _NSConcreateMallocBlock ( 保存在堆中的 block，当引用计数为 0 时会被销毁)

 ![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-20-051428.jpg)

#### _NSConcreteGlobalBlock

未捕获任何变量或仅捕获的变量为以下类型的 Block 是 NSConcreteGlobalBlock。

- 静态变量
- 全局变量
- 静态全局变量



```objective-c
void (^blk) () = ^{
    printf("Block");
};
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        blk();
    }
    return 0;
}
```

全局定义的Block经过代码转换后

```objective-c
struct __blk_block_impl_0 {
  struct __block_impl impl;
  struct __blk_block_desc_0* Desc;
  __blk_block_impl_0(void *fp, struct __blk_block_desc_0 *desc, int flags=0) {
    impl.isa = &_NSConcreteGlobalBlock; //设置在数据区的Block
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
```



#### _NSConcreteStackBlock

只要捕获了以上三种类型以外的变量的 Block 是 NSConcreteStackBlock。

```objective-c
int i = 0;
void (^blk)() = ^() {
    printf("%d", i);
};
blk();
```

局部定义的Block 经过代码转换后

```objective-c
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) {
    impl.isa = &_NSConcreteStackBlock; //设置在栈区的Block
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
```

大多数时候，clang转换的源代码通常是`_NSConcreteStackBlock`对象

**可以这么理解，NSConcreteStackBlock就是引用了外部变量的block**

#### NSConcreteMallocBlock

系统不提供直接创建 `NSConcreteMallocBlock` 的方式，但是可以对 `NSConcreteStackBlock` 进行 copy 操作来生成 `NSConcreteMallocBlock`

以下情况，Block 会进行 copy 操作：

- 手动执行 copy 方法
- 将 Block 赋值给 __strong 修饰符修饰（系统默认）的 Block 或者 id 对象
- 作为方法的返回值
- 系统 API 中含有 usingBlcok 的方法


![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-20-051433.jpg)

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-20-051439.jpg)



### Block生命周期

`NSConcreteStackBlock` 是由编译器自动管理，超过作用域之外就会自动释放了。而 `NSConcreteMallocBlock` 是由程序员自己管理，如果没有被强引用也会被消耗。`NSConcreteGlobalBlock` 由于存在于全局区，所以会一直伴随着应用程序。



## __forwarding存在意义

不论在任何内存位置,都可以顺利访问同一个__block变量.

## Reference

[1.深入理解 Block](https://xiaozhuanlan.com/topic/2710695843)