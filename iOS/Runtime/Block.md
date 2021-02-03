#   Block

## Block构造

> In programming languages, a closure is a function or reference to a function together with a referencing environment—a table storing a reference to each of the non-local variables (also called free variables or upvalues) of that function.

* `Block` 是将函数及其执行上下文封装起来的`对象`
* `Block`的调用即是函数的调用

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



## Block 写法

```objective-c

@property (nonatomic, copy)void (^addBlockResult)(BOOL) ;

int multiplier = 6
int(^Block)(int) = ^int(int num){
    return num * multiplier;
}
```



## Clang 重写Block

### 1.Block捕获外部变量实质

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



```objective-c
#import <Foundation/Foundation.h>

int global_i = 1;

static int static_global_j = 2;

int main(int argc, const char * argv[]) {
   
    static int static_k = 3;
    int val = 4;
    
    void (^myBlock)(void) = ^{
        global_i ++;
        static_global_j ++;
        static_k ++;
        NSLog(@"Block中 global_i = %d,static_global_j = %d,static_k = %d,val = %d",global_i,static_global_j,static_k,val);
    };
    
    global_i ++;
    static_global_j ++;
    static_k ++;
    val ++;
    NSLog(@"Block外 global_i = %d,static_global_j = %d,static_k = %d,val = %d",global_i,static_global_j,static_k,val);
    
    myBlock();
    
    return 0;
}
```

运行结果

```vim
Block 外  global_i = 2,static_global_j = 3,static_k = 4,val = 5
Block 中  global_i = 3,static_global_j = 4,static_k = 5,val = 4
```

1.为什么在Block里面不加__bolck不允许更改变量？
2.为什么自动变量的值没有增加，而其他几个变量的值是增加的？自动变量是什么状态下被block捕获进去的？



```objective-c
int global_i = 1;

static int static_global_j = 2;

struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  int *static_k;
  int val;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int *_static_k, int _val, int flags=0) : static_k(_static_k), val(_val) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
  int *static_k = __cself->static_k; // bound by copy
  int val = __cself->val; // bound by copy

        global_i ++;
        static_global_j ++;
        (*static_k) ++;
        NSLog((NSString *)&__NSConstantStringImpl__var_folders_45_k1d9q7c52vz50wz1683_hk9r0000gn_T_main_6fe658_mi_0,global_i,static_global_j,(*static_k),val);
    }

static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};


int main(int argc, const char * argv[]) {

    static int static_k = 3;
    int val = 4;

    void (*myBlock)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, &static_k, val));

    global_i ++;
    static_global_j ++;
    static_k ++;
    val ++;
    NSLog((NSString *)&__NSConstantStringImpl__var_folders_45_k1d9q7c52vz50wz1683_hk9r0000gn_T_main_6fe658_mi_1,global_i,static_global_j,static_k,val);

    ((void (*)(__block_impl *))((__block_impl *)myBlock)->FuncPtr)((__block_impl *)myBlock);

    return 0;
}
```

首先全局变量global_i和静态全局变量static_global_j的值增加，以及它们被Block捕获进去，这一点很好理解，因为是全局的，作用域很广，所以Block捕获了它们进去之后，在Block里面进行++操作，Block结束之后，它们的值依旧可以得以保存下来。



## Reference

[1 深入研究 Block 捕获外部变量和 __block 实现原理](https://halfrost.com/ios_block/)

