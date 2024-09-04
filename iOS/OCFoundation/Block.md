

#   Block

## 1. Block构造

> In programming languages, a closure is a function or reference to a function together with a referencing environment—a table storing a reference to each of the non-local variables (also called free variables or upvalues) of that function.

* `Block` 是将函数及其执行上下文封装起来的`对象`
* `Block`的调用即是函数的调用
* `Block`本质上也是一个OC对象，它内部也有个isa指针
* `Block`是封装了函数调用以及函数调用环境的OC对象

Block在OC中的实现如下：

```objective-c
struct Block_layout {
    void *isa;
    int flags;
    int reserved;
    void (*invoke)(void *, ...);
    struct Block_descriptor *descriptor;
    /* Imported variables. */
};

struct Block_descriptor {
    unsigned long int reserved;
    unsigned long int size;
    void (*copy)(void *dst, void *src);
    void (*dispose)(void *);
};
```

![img](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-02-03-082510.png)

从结构图中很容易看到isa，所以OC处理Block是按照对象来处理的。在iOS中，isa常见的就是`_NSConcreteStackBlock`，`_NSConcreteMallocBlock`，`_NSConcreteGlobalBlock`这3种



## 2. Block 写法

```objective-c

@property (nonatomic, copy)void (^addBlockResult)(BOOL) ;

int multiplier = 6
int(^Block)(int) = ^int(int num){
    return num * multiplier;
}
```



## 3. Clang 重写Block

### Block捕获外部变量实质

说到外部变量，我们要先说一下C语言中变量有哪几种。一般可以分为一下5种：

- 自动变量
- 函数参数
- 静态变量
- 静态全局变量
- 全局变量

研究Block的捕获外部变量就要除去函数参数这一项，下面一一根据这4种变量类型的捕获情况进行分析。

我们先根据这4种类型

- 自动变量
- 静态变量
- 静态全局变量
- 全局变量





如下代码

```objective-c
int age = 20;
void (^block)(void) =  ^{
     NSLog(@"age is %d",age);
 };
        
block();

```

- 打开终端，cd到当前目录下

> xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc main.m

生成`main.cpp`

#### block 结构分析

```cpp
int age = 20;

// block的定义
void (*block)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, age));
// block的调用
((void (*)(__block_impl *))((__block_impl *)block)->FuncPtr)((__block_impl *)block);

```

上面的代码删除掉一些强制转换的代码就就剩下如下所示

```cpp
int age = 20;
void (*block)(void) = &__main_block_impl_0(
						__main_block_func_0, 
						&__main_block_desc_0_DATA, 
						age
						);
// block的调用
block->FuncPtr(block);

```





**看出block的本质就是一个结构体对象**，结构体`__main_block_impl_0`代码如下





```cpp
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  int age;
    //构造函数(类似于OC中的init方法) _age是外面传入的
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int _age, int flags=0) : age(_age) {
    //isa指向_NSConcreteStackBlock 说明这个block就是_NSConcreteStackBlock类型的
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};

```



结构体中第一个是`struct __block_impl impl;`

```cpp
struct __block_impl {
      void *isa;
      int Flags;
      int Reserved;
      void *FuncPtr;
};       

```



结构体中第二个是`__main_block_desc_0;`

```cpp
static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size; // 结构体__main_block_impl_0 占用的内存大小
}

```

结构体中第三个是`age`



也就是捕获的局部变量 `age`

```cpp
__main_block_func_0
//封装了block执行逻辑的函数
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
    int age = __cself->age; // bound by copy
    
    NSLog((NSString *)&__NSConstantStringImpl__var_folders_x4_920c4yq936b63mvtj4wmb32m0000gn_T_main_7f3f1b_mi_0,age);
}
```

用一幅图来表示

![img](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-02-04-151812.jpg)



### 变量捕获

其实上面的代码我们已经看得出来变量捕获了，这里继续详细分析一下

| 变量类型        | 捕获到block内部 | 访问方式 |
| --------------- | --------------- | -------- |
| 局部变量 auto   | √               | 值传递   |
| 局部变量 static | √               | 指针传递 |
| 全局变量        | ×               | 直接访问 |





### 3.1 局部变量auto(自动变量)

- 我们平时写的局部变量，默认就有 auto(自动变量，离开作用域就销毁)

##### 运行代码

例如下面的代码

```objective-c
int age = 20;
void (^block)(void) =  ^{
     NSLog(@"age is %d",age);
};
age = 25;
       
block();
```

等同于

```objective-c
auto int age = 20;
void (^block)(void) =  ^{
     NSLog(@"age is %d",age);
};
age = 25;
       
block();
```

输出

> 20

##### 分析

> xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc main.m

生成`main.cpp`

如图所示

![img](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-02-04-152053.jpg)



```cpp
int age = 20;
void (*block)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, age));
age = 25;

((void (*)(__block_impl *))((__block_impl *)block)->FuncPtr)((__block_impl *)block);

struct __main_block_impl_0 *blockStruct = (__bridge struct __main_block_impl_0 *)block;

NSLog((NSString *)&__NSConstantStringImpl__var_folders_x4_920c4yq936b63mvtj4wmb32m0000gn_T_main_d36452_mi_5);

```



可以知道，直接把age的值 20传到了结构体`__main_block_impl_0`中，后面再修改`age = 25`并不能改变block里面的值





### 3.2 局部变量 static

static修饰的局部变量，不会被销毁

##### 运行代码

eg

```objective-c
static int height  = 30;
int age = 20;
void (^block)(void) =  ^{
     NSLog(@"age is %d height = %d",age,height);
};
age = 25;
height = 35;
block();
```

执行结果为

```shell
age is 20 height = 35
```

可以看得出来，block外部修改height的值，依然能影响block内部的值

##### 析

> xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc main.m

生成`main.cpp`



![img](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-02-04-152048.jpg)



```cpp
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
  int age = __cself->age; // bound by copy
  int *height = __cself->height; // bound by copy

            NSLog((NSString *)&__NSConstantStringImpl__var_folders_x4_920c4yq936b63mvtj4wmb32m0000gn_T_main_3146e1_mi_4,age,(*height));
        }

static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};
int main(int argc, const char * argv[]) {
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 



        static int height = 30;
        int age = 20;
        void (*block)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, age, &height));
        age = 25;
        height = 35;
        ((void (*)(__block_impl *))((__block_impl *)block)->FuncPtr)((__block_impl *)block);

```

如图所示，`age`是直接值传递，`height`传递的是`*height` 也就是说直接把内存地址传进去进行修改了。





### 3.3 全局变量

##### 运行代码

```objective-c
int age1 = 11;
static int height1 = 22;

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        void (^block)(void) =  ^{
            NSLog(@"age1 is %d height1 = %d",age1,height1);
        };
        age1 = 25;
        height1 = 35;
        block();

    }
    return 0;
}
```

输出结果为

```shell
age1 is 25 height1 = 35
```

##### 分析

> xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc main.m

生成`main.cpp`



![img](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-02-04-152316.jpg)



```cpp
struct __main_block_impl_0 {
    struct __block_impl impl;
    struct __main_block_desc_0* Desc;
    __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) {
        impl.isa = &_NSConcreteStackBlock;
        impl.Flags = flags;
        impl.FuncPtr = fp;
        Desc = desc;
    }
};
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
    
    NSLog((NSString *)&__NSConstantStringImpl__var_folders_x4_920c4yq936b63mvtj4wmb32m0000gn_T_main_4e8c40_mi_4,age1,height1);
}

static struct __main_block_desc_0 {
    size_t reserved;
    size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};
int main(int argc, const char * argv[]) {
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 

        void (*block)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA));
        age1 = 25;
        height1 = 35;
        ((void (*)(__block_impl *))((__block_impl *)block)->FuncPtr)((__block_impl *)block);
        
    }
    return 0;
}
```

从cpp文件可以看出来，并没有捕获全局变量age1和height1,访问的时候，是直接去访问的，根本不需要捕获



#### 小结

| 变量类型        | 捕获到block内部 | 访问方式 |
| --------------- | --------------- | -------- |
| 局部变量 auto   | √               | 值传递   |
| 局部变量 static | √               | 指针传递 |
| 全局变量        | ×               | 直接访问 |

- auto修饰的局部变量，是值传递
- static修饰的局部变量，是指针传递

其实也很好理解，因为auto修饰的局部变量，离开作用域就销毁了。那如果是指针传递的话，可能导致访问的时候，该变量已经销毁了。程序就会出问题。而全局变量本来就是在哪里都可以访问的，所以无需捕获。



## 4  block有3种类型

#### block也是一个OC对象

block有3种类型，可以通过调用class方法或者isa指针查看具体类型，最终都是继承自NSBlock类型

- `__NSGlobalBlock__ （ _NSConcreteGlobalBlock ）` (全局的静态 block，不会访问任何外部变量)
- `__NSStackBlock__ （ _NSConcreteStackBlock ）` ( 保存在栈中的 block，当函数返回时会被销毁)
- `__NSMallocBlock__ （ _NSConcreteMallocBlock ）` ( 保存在堆中的 block，当引用计数为 0 时会被销毁)

其中三种不同的类型和环境对应如下

| block类型           | 环境                         |
| ------------------- | ---------------------------- |
| `__NSGlobalBlock__` | 没有访问auto变量             |
| `__NSStackBlock__`  | 访问了auto变量               |
| `__NSMallocBlock__` | `__NSStackBlock__`调用了copy |

每一种类型的 block 调用 copy 后的结果如下所示

|        Block 的类        | 副本源的配置存储域 |   复制效果   |
| :----------------------: | :----------------: | :----------: |
| `_NSConcreteGlobalBlock` |   程序的数据区域   |  什么也不做  |
| `_NSConcreteStackBlock`  |         栈         | 从栈复制到堆 |
| `_NSConcreteMallocBlock` |         堆         | 引用计数增加 |



在 ARC 环境下，编译器会根据情况自动将栈上的 block 复制到堆上，比如以下情况：

- block 作为函数返回值时
- 将 block 赋值给 __strong 指针时
- block 作为 Cocoa API 中方法名含有 using Block 的方法参数时
- Block 作为 GCD APIE 的方法参数时



![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-20-051428.jpg)

### 5 Block生命周期

`NSConcreteStackBlock` 是由编译器自动管理，超过作用域之外就会自动释放了。而 `NSConcreteMallocBlock` 是由程序员自己管理，如果没有被强引用也会被消耗。`NSConcreteGlobalBlock` 由于存在于全局区，所以会一直伴随着应用程序。

无论是MAC还是ARC

- 当block为`__NSStackBlock__`类型时候，是在栈空间，无论对外面使用的是strong 还是weak 都不会对外面的对象进行强引用
- 当block为`__NSMallocBlock__`类型时候，是在堆空间，block是内部的`_Block_object_assign`函数会根据`strong`或者 `weak`对外界的对象进行强引用或者弱引用。

其实也很好理解，因为block本身就在栈上，自己都随时可能消失，怎么能保住别人的命呢？

- 当block内部访问了对象类型的auto变量时
- 如果block是在栈上，将不会对auto变量产生强引用
- 如果block被拷贝到堆上
  - 会调用block内部的copy函数
  - copy函数内部会调用`_Block_object_assign`函数
  - `_Block_object_assign`函数会根据auto变量的修饰符`（__strong、__weak、__unsafe_unretained）`做出相应的操作，形成强引用（retain）或者弱引用
- 如果block从堆上移除
  - 会调用block内部的dispose函数
  - dispose函数内部会调用`_Block_object_dispose`函数
  - `_Block_object_dispose`函数会自动释放引用的auto变量（release）

| 函数        | 调用时机              |
| ----------- | --------------------- |
| copy函数    | 栈上的Block复制到堆上 |
| dispose函数 | 堆上的block被废弃时   |





weak的实现原理，在原对象释放之后，weak对象就会变成null，防止野指针。所以就输出了null了。

那么我们怎么才能在weakSelf之后，block里面还能继续使用weakSelf之后的对象呢？

究其根本原因就是weakSelf之后，无法控制什么时候会被释放，为了保证在block内不会被释放，需要添加_strong。

在block里面使用的_strong修饰的weakSelf是为了在函数生命周期中防止self提前释放。strongSelf是一个自动变量当block执行完毕就会释放自动变量strongSelf不会对self进行一直进行强引用。







## 6 __block 修饰符

**__block修饰符原理：**

编译器会将`__block`变量包装成一个结构体`__Block_byref_age_0`，结构体内部`*__forwarding`是指向自身的指针，内部还存储着外部`auto变量`的值

一开始，栈空间的block有一个`__Block_byref_a_0`结构体，
 指向外部`__Block_byref_a_0`的地址，
 其中它的__forwarding指针指向自身，

当block从栈copy到堆时，

堆空间的block有一个`__Block_byref_a_0`结构体，
 指向外部`__Block_byref_a_0`的地址，
 其中它的__forwarding指针指向自身





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

### 6.1 __block原理

1. 当__block修饰外界变量时

```
int main(){
    
    __block int a = 10;
    void(^block)(void) = ^{
        printf("Felix %d ", a);
    };
    
    block();
    return 0;
}
```

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-02-14-084559.jpg)



#### 将代码编译成C++源码



```objective-c
// 原代码
__block int a = 10;
// c++源码
__attribute__((__blocks__(byref))) __Block_byref_a_0 a = {
    (void*)0,
    (__Block_byref_a_0 *)&a, 
    0, 
    sizeof(__Block_byref_a_0), 
    10
};
```

可以看到 变量a 变成了 结构体类型`__Block_byref_a_0`

下面再看看结构体`__Block_byref_a_0`的构造



```cpp
struct __Block_byref_a_0 {
  void *__isa;
__Block_byref_a_0 *__forwarding;
 int __flags;
 int __size;
 int a;
};
```

通过上面结构体的初始化和结构体的构造，
 可以获得以下信息：

> 1. __forwarding存放的是自己本身的地址
> 2. 结构体内的a变量存放的是外部变量a的值

主结构体`__main_block_impl_0`的变化

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-02-14-085104.jpg)

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-02-14-085118.jpg)



## 如何从栈指向堆，并建立联系呢？

apple源码，如图：

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-02-14-085605.jpg)



copy->forwarding = copy;
 就是将堆结构体的__forwarding指针指向自身
 src->forwarding = copy;
 就是将栈结构体的__forwarding指针指向堆结构体

这样，苹果工程师在背后悄悄地将block copy到了堆上，
 而且栈上的block从未被我们利用过。

在看看block入口静态函数



```rust
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
  __Block_byref_a_0 *a = __cself->a; // bound by ref
  (a->__forwarding->a)++;
}
```

通过当前栈空间主结构体上的`__Block_byref_a_0`结构体指针，访问指向堆空间的`__forwarding`成员，并获取堆空间上变量的值。

**当然，不仅__block修饰的变量会这样，前文的对象类型变量同样会在copy函数内部被转化成类似的结构体进行处理。**





`__block`修饰的属性在底层会生成响应的结构体，保存原始变量的指针，并传递一个指针地址给block——因此是指针拷贝



`__block` 所起到的作用就是只要观察到该变量被 block 所持有，就将“外部变量”在栈中的内存地址放到了堆中。进而在block内部也可以修改外部变量的值。

__block修饰的变量成了对象

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-20-051401.jpg)

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-20-051409.jpg)

**栈上**的 \__block 的  ` __forwading` 指针指向自己



![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-23-035859.jpg)



### 6.3  __forwarding存在意义

不论在任何内存位置,都可以顺利访问同一个__block变量.

## Reference

[1 深入研究 Block 捕获外部变量和 __block 实现原理](https://halfrost.com/ios_block/)

[2 深入理解iOS的block](https://juejin.cn/post/6844903893176958983)

[3 iOS中__block 关键字的底层实现原理](https://www.jianshu.com/p/404ff9d3cd42)

[4 iOS探索 全方位解读Block](https://juejin.cn/post/6844904181778612231)

[5 iOS - block原理解读（三）](https://www.jianshu.com/p/9af777c7d222)

[探寻Block的本质（6）—— __block的深入分析__block的使用场景 大家应该都知道，如果想在block - 掘金 (juejin.cn)](https://juejin.cn/post/6968314665667395620)