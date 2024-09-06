

#   Block



##  Block本质

> In programming languages, a closure is a function or reference to a function together with a referencing environment—a table storing a reference to each of the non-local variables (also called free variables or upvalues) of that function.



![img](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-02-03-082510.png)





* `Block` 是将函数及其执行上下文封装起来的`对象`
* `Block`的调用即是函数的调用
* `Block`本质上也是一个OC对象，它内部也有个isa指针
* `Block`是封装了函数调用以及函数调用环境的OC对象



block地层结构图中的第一个成员就是一个isa指针，所以我们可以将block当成一个对象来看待。isa常见的就是`_NSConcreteStackBlock`，`_NSConcreteMallocBlock`，`_NSConcreteGlobalBlock`这3种



## Block 写法

```objective-c

@property (nonatomic, copy)void (^addBlockResult)(BOOL) ;

int multiplier = 6
int(^Block)(int) = ^int(int num){
    return num * multiplier;
}
```



## Block 结构

```objective-c
//如下代码
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        int a = 10;
        void (^block)(int, int) = ^(int c, int b){
            NSLog(@"I am a block!");
            NSLog(@"I am a block!");
            NSLog(@"c = %d",c);
            NSLog(@"b = %d",b);
            NSLog(@"a的值为%d",a);
        };
        block(50,100);
    }
    return 0;
}
```

通过

```shell
 xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc main.m -o main.cpp
```

将OC文件用Clang重写

```objective-c
#import <Foundation/Foundation.h>

//将block的底层结构struct __main_block_impl_0直接般到main.m里面

struct __block_impl {
    void *isa;
    int Flags;
    int Reserved;
    void *FuncPtr;
};

struct __main_block_desc_0 {
    size_t reserved;
    size_t Block_size;
};

struct __main_block_impl_0 {
    struct __block_impl impl;
    struct __main_block_desc_0* Desc;
    int a;
    
};

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        
        int a = 10;
        
        void (^block)(int, int) = ^(int c, int b){
            NSLog(@"I am a block!");
            NSLog(@"I am a block!");
            NSLog(@"c = %d",c);
            NSLog(@"b = %d",b);
            NSLog(@"a的值为%d",a);
            
        };
        
        struct __main_block_impl_0 *tmpBlock = (__bridge struct __main_block_impl_0 *)block;
        
        block(50,100);
        

    }
    return 0;
}

```

![运行时下block内部的信息](https://sylarimage.oss-cn-shenzhen.aliyuncs.com/uPic/232592486bdb48339722e47dde38b11c~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)



### Block 底层代码



![img](https://sylarimage.oss-cn-shenzhen.aliyuncs.com/uPic/47de259461274c8d8e83ce3afa7f5d71~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)



## Block 捕获外部变量



### Block 捕获基础类型



#### Block捕获auto变量

接下来看看这种情况

```Objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        int age = 10;
        //Block的定义
        void (^block)(void) = ^(){
            NSLog(@"Age is %d", age);
        };
        //先修改age的值
        age = 20;
        //Block的调用
        block();
    }
    return 0;
}
```



在block之前定义了一个` int a = 10`，然后在block内部使用了这个`age`，而且我在调用block之前，先将`age`的值修改成了`20`，那么此时程序运行会是什么结果呢



```c
Interview03-block[4064:375528] Age is 10
Program ended with exit code: 0
```



结果是block中打印出的`a`是`10`，我们在block外部对`age`的修改结果并没有对block的内部打印产生影响



**(1)首先看一下此时block对应的结构体**

 ![img](https://sylarimage.oss-cn-shenzhen.aliyuncs.com/uPic/5c029bf36ba947bb9d5a273384e93756~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)



我们发现有三处变化

- 新增了一个`int age`成员变量
- 构造函数里面多了一个参数 `int _age`
- 构造函数里面参数尾部多了一个`: age(_age)`，这是c++的语法，作用时将参数`_age`自动赋值给成员变量`age`





**(2)再看一下main函数中的block定义以及赋值的代码**

![img](https://sylarimage.oss-cn-shenzhen.aliyuncs.com/uPic/99f0f5408908484c8b75a79073ed7a06~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)



在用`block`构造函数生成`block`的时候，使用了外部定义的 `int a = 10`,因为c函数的参数都是值传递，所以这里是将此时外部变量`a`的值`10`传给了`block`的构造函数`__main_block_impl_0`，因此block内部的成员变量`age`会被赋值成`10`。



**(3)再看一下block内部封装的函数**![img](https://sylarimage.oss-cn-shenzhen.aliyuncs.com/uPic/fcab6d4d243e467581f1b89aadfa6a4f~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

可以看到打印代码里面使用的`age`，实际上就是block内部的成员变量`age`，不是我们在外面定义的那个`age`，因此，当block被赋值之后，其成员变量`age`被赋值成了当时构造函数传进来的参数`10`，所以最终打印出来值就是`10`，**不论外部的`age`再如何的修改。外部的`age`跟block的成员变量`age`是两个不同的变量，互不影响。**



其实，上面我门讨论的这个block外部变量`age`是一个**局部auto变量**，也叫**自动变量**。除了`auto变量`，C语言里面还有局部`static变量`（**静态变量**）和**全局变量**，接下来我们就看看，**Block**对于这几种变量的使用，做了如何的处理。





#### Block捕获局部static变量

首先我们将上面的OC代码改造如下

```Objc
#import <Foundation/Foundation.h>

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        int age = 10;
        static int height = 10;
        //Block的定义
        void (^block)(void) = ^(){
            NSLog(@"Age is %d, height is %d", age, height);
        };
        //先修改age和height的值
        age = 20;
        height = 20;
        //Block的调用
        block();
    }
    return 0;
}
```

我们有增加了一个`static`变量`height`，并且在同样的地方修改`height`的值，便于和之前的`age`进行对比。首先运行代码看一下结果

```c
Interview03-block[4725:476530] Age is 10, height is 20
Program ended with exit code: 0
```

可以看到，block输出的 `height`值是我们在外部重新为其赋的`20`。 



**(1)借用上面的分析流程一样，先看一下block对应的结构体**

![img](https://sylarimage.oss-cn-shenzhen.aliyuncs.com/uPic/609162bfd8cd485180b756c2e76cc3ad~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)



针对static变量height, block内部为其增加了一个`int *height;`成员变量，构造函数里面对应的参数是`int *_height`。看到这里这里要存储的是一个地址，该地址应该就是外部`static`变量`height`的地址值。



**(2)main函数里的block赋值过程**



![对于static变量，block捕获的是它的地址](https://sylarimage.oss-cn-shenzhen.aliyuncs.com/uPic/23a8d3d5111b4931a9e1fa8f2839ff0e~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

block构造函数里面传入的，就是外部的这个height的地址值。



**(3)block内部的函数**



![img](https://sylarimage.oss-cn-shenzhen.aliyuncs.com/uPic/4a95ec27fd494cadb3d8445563438c36~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)



那么可以看到，block内部的函数也是通过block所存储的地址值`*height`访问了外部的`static`变量`height`的值。

因此，当我们从外部修改`height`的值之后，调用block打印出的`height`的值也相应的改变了，因为block内部是通过 **指针** 引用了外部的这个`static`变量`height`。



#### 对于`auto`、`static`变量，为什么**block**选择用不同方式处理它们呢？

一个**自动变量**（`auto`）的存储空间位于函数栈空间上，在函数开辟栈空间时被创建，在函数结束时销毁，而**block**的调用时机有可能发生在函数结束之后的，因此就无法使用自动变量了，所以在**block**一开始定义赋值的过程里，就将自动变量的值拷贝到他自己的存储空间上。 而对于局部静态变量（`static`），C语法下`static`会改变所修饰的局部变量的生命周期，使其在  **程序整个运行期间都存在**  ，所以**block**选择持有它的指针，在**block**被调用时，通过该指针访问这个变量的内容就行。



#### Block使用全局变量

上面讨论block对于局部变量的处理，在看一看对于全局变量，情况又是如何

![img](https://sylarimage.oss-cn-shenzhen.aliyuncs.com/uPic/0f909ae551b5422daa0d833fd61515fc~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)



输出结果如下

```c
Interview03-block[13997:1263406] Age is 20, height is 20
Program ended with exit code: 0
```



在通过命令行生成一下编译后的C++文件，同样还是在文件底部去看

![block使用全局变量](https://sylarimage.oss-cn-shenzhen.aliyuncs.com/uPic/36bc8f341ec8400cbe3b146a272f0374~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)



**block**没有对全局变量进行捕获行为，只需要在要用的时候，直接通过变量名访问就行了，因为全局变量时跨函数的，可以直接通过变量的名字直接访问。 同样，者也帮我我们理解了为什么对于局部的变量，**block**需要对其采取“捕获”行为，正是因为局部变量定在与函数内部，无法跨函数使用，所以根据局部变量不同的存储属性，要么将其值直接进行拷贝（`auto`），要么对其地址进行拷贝（`static`)。



#### 总结



![img](https://sylarimage.oss-cn-shenzhen.aliyuncs.com/uPic/4f3765d51d9e4b958d9fa7de0e96f8fe~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

1. **局部变量会被block捕获**

- 自动变量（`auto`），**block**通过值拷贝方式捕获，在其内部创建一个同类型变量，并且将自动变量的值拷贝给**block**的内部变量，**block**代码块执行的时候，直接访问它的这个***内部变量***。
- 静态变量（`static`），**block**通过地址拷贝方式捕获，在其内部创建一个指向同类型变量的指针， 将静态变量的地址值拷贝给**block**内部的这个指针，**block**代码块执行的时候，通过内部存储的指针***间接访问静态变量***。

1. **全局变量不会被block捕获**,    block代码块执行的时候，通过全局变量名***直接访问***。



#### Block对于self的处理



![img](https://sylarimage.oss-cn-shenzhen.aliyuncs.com/uPic/a2899927dade46ad8cee33769a27da14~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) 



![img](https://sylarimage.oss-cn-shenzhen.aliyuncs.com/uPic/52077c6863b94f299038ed875a0174d3~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)



编译结果显示**block**对`self`进行了捕获。But why？ 我们知道，图中的**block**位于`test`方法里面，实际上任何的oc方法，转换成底层的c函数，里面都有两个默认

的参数，`self` 和 `_cmd`



![img](https://sylarimage.oss-cn-shenzhen.aliyuncs.com/uPic/7d7b27a9efb34d6db729048a3ca7aa79~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)



所以作为函数默认参数的`self`的实际上也是该函数的局部变量，根据我们上面总结的原则，只要是局部变量，**block**都会对其进行捕获，这就解释通了。





下面的情况呢



![img](https://sylarimage.oss-cn-shenzhen.aliyuncs.com/uPic/eecbec499b74418bb3a740c453e30440~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) 



先看编译结果



![img](https://sylarimage.oss-cn-shenzhen.aliyuncs.com/uPic/9b6f09a9b4d94d4eb9d0965c5f89ce22~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)



看得出来，还是进行了捕获，在图中标明的黄色框框，就很好理解了，**block**最终访问`CLPerson`的成员变量`_age`的时候，是通过`self` +` _age偏移量`，获得`_age`的地址后从而进行间接访问的，所以在oc代码中，`_age` 的写法等同与`self->_age`，说白了，这里还是需要用到`self`，因此**block**还是需要对`self`进行捕获的。



### Block捕获对象类型



有以下代码:

```objective-c
typedef void(^CLBlock)(void);//➕➕➕

int main(int argc, const char * argv[]) {
    @autoreleasepool {
	
		CLBlock myBlock;
        {//临时作用域开始
            CLPerson *person = [[CLPerson alloc] init];
            person.age = 10;

			 myBlock = ^{
                NSLog(@"---------%d",person.age);
            };
        }//临时作用域结束
        
        NSLog(@"-----------flag1");
    }
    return 0;
}

```

由于现在是**ARC**环境，`myBlock`属于强指针，因此在将**block**对象赋值给`myBlock`指针的时候，编译器会自动对**block**对象执行`copy`操作，因此赋值完成后，`myBlock`指向的是一个堆空间上的**block**对象副本

通过Clang重写

```c++
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  CLPerson *person;

  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, CLPerson *_person, int flags=0) : person(_person) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};


static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
  CLPerson *person = __cself->person; // bound by copy

                NSLog((NSString *)&__NSConstantStringImpl__var_folders_7__p19yp82j0xd2m_1k8fpr77z40000gn_T_main_2cca58_mi_0,((int (*)(id, SEL))(void *)objc_msgSend)((id)person, sel_registerName("age")));
            }



static void __main_block_copy_0(struct __main_block_impl_0*dst, struct __main_block_impl_0*src) {
            _Block_object_assign((void*)&dst->person, (void*)src->person, 3/*BLOCK_FIELD_IS_OBJECT*/);
}




static void __main_block_dispose_0(struct __main_block_impl_0*src) {
            _Block_object_dispose((void*)src->person, 3/*BLOCK_FIELD_IS_OBJECT*/);
}




static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
  void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);
  void (*dispose)(struct __main_block_impl_0*);
} __main_block_desc_0_DATA = { 
                                0, 
                                sizeof(struct __main_block_impl_0), 
                                __main_block_copy_0, 
                                __main_block_dispose_0
                              };



int main(int argc, const char * argv[]) {
    /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 
        CLBlock myBlock;

        {
            CLPerson * person = objc_msgSend(objc_msgSend(objc_getClass("CLPerson"), 
                                                          sel_registerName("alloc")
                                                          ), 
                                             sel_registerName("init")
                                             );
            

            objc_msgSend(person, 
                         sel_registerName("setAge:"), 
                         30
                         );


            myBlock = objc_msgSend(&__main_block_impl_0(__main_block_func_0, 
                                                        &__main_block_desc_0_DATA, 
                                                        person, 
                                                        570425344), 
                                   sel_registerName("copy")
                                   );


        }


    }


    return 0;
}

```



`__main_block_desc_0`结构体里面多了两个彩蛋

- 函数指针`copy`，也就是`__main_block_copy_0()`，内部调用了`_Block_object_assign()`
- 函数指针`dispose`，也就是`__main_block_dispose_0()`，内部调用了`_Block_object_dispose()`



![捕捉对象类型的auto变量时__main_block_desc_0的变化](https://sylarimage.oss-cn-shenzhen.aliyuncs.com/uPic/fa7895e1bbd44546a26aba3723be24df~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)



**ARC** 下`CLPerson *person`被认为是强指针，等价于`_strong CLPerson *person`，而弱指针需要显式地表示为`__weak CLPerson *person`。通过终端命令`xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc -fobjc-arc -fobjc-runtime=ios-9.0.0 main.m -o main.cpp`，可以看到`block`的内捕获到的`person`指针如下

![image](https://sylarimage.oss-cn-shenzhen.aliyuncs.com/uPic/8d135f2bf3b94f25ad4358f7ae63f9f6~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)



为了对比，我们再分别看一下下面三种 场景分别是什么情况的：

- **ARC环境-->堆上的`block`-->弱指针`__weak CLPerson *person`**
- **ARC环境-->栈上的`block`-->强指针`CLPerson *person`**
- **ARC环境-->栈上的`block`-->弱指针`__weak CLPerson *person`**



【**ARC环境-->堆上的`block`-->弱指针`__weak CLPerson *person`**】 案例如下

```c
***********************main.m*************************
#import <Foundation/Foundation.h>
#import "CLPerson.h"

typedef void(^CLBlock)(void);


int main(int argc, const char * argv[]) {
    @autoreleasepool {
        CLBlock myBlock;
        
        {//临时作用域开始
            __weak CLPerson * person = [[CLPerson alloc] init];
            person.age = 30;
            
            myBlock = ^{
                NSLog(@"---------%d",person.age);
            } ;
            
        }//临时作用域结束

        NSLog(@"-------------");
        
    }
    
    NSLog(@"------main autoreleasepool end-------");
    
    return 0;
}
```

block的底层结构如下

 ![image](https://sylarimage.oss-cn-shenzhen.aliyuncs.com/uPic/672cb3ea23714ba097c0d1ea73f7a172~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) 



![ARC->堆block->弱指针运行结果](https://sylarimage.oss-cn-shenzhen.aliyuncs.com/uPic/303a83e7baae4b9cada6abf22e4cd35f~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) 



运行结果显示堆上的**block**使用弱指针`__weak CLPerson *person`，没有影响`person`所指向对象的生命周期，出了临时作用域的之后就被释放了。



【**ARC环境-->栈上的`block`-->强指针`CLPerson *person`**】

```c
***********************main.m*************************
#import <Foundation/Foundation.h>
#import "CLPerson.h"

typedef void(^CLBlock)(void);


int main(int argc, const char * argv[]) {
    @autoreleasepool {
        CLBlock myBlock;
        
        {//临时作用域开始
            CLPerson * person = [[CLPerson alloc] init];
            person.age = 30;
            
            ^{
                NSLog(@"---------%d",person.age);
            } ;
            
        }//临时作用域结束

        NSLog(@"-------------");
        
    }
    
    NSLog(@"------main autoreleasepool end-------");
    
    return 0;
}
```

block底层结构如下 ![image](https://sylarimage.oss-cn-shenzhen.aliyuncs.com/uPic/f8226f12e1594970ab13d2d15c5538d3~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)



![ARC->栈block->强指针运行结果](https://sylarimage.oss-cn-shenzhen.aliyuncs.com/uPic/a1f3099ad5054136b6d28c56e00c8398~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

运行结果显示栈上的**block**使用强指针` CLPerson *person`，没有影响`person`所指向对象的生命周期，出了临时作用域的之后就被释放了。



【**ARC环境-->栈上的`block`-->弱指针`__weak CLPerson *person`**】

```c
***********************main.m*************************
#import <Foundation/Foundation.h>
#import "CLPerson.h"

typedef void(^CLBlock)(void);


int main(int argc, const char * argv[]) {
    @autoreleasepool {
        CLBlock myBlock;
        
        {//临时作用域开始
            __weak CLPerson * person = [[CLPerson alloc] init];
            person.age = 30;
            
            ^{
                NSLog(@"---------%d",person.age);
            } ;
            
        }//临时作用域结束

        NSLog(@"-------------");
        
    }
    
    NSLog(@"------main autoreleasepool end-------");
    
    return 0;
}
```

block底层结构为 

![image](https://sylarimage.oss-cn-shenzhen.aliyuncs.com/uPic/0f02a2aa0ae940a09ca40935bc5ef8c1~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)



![ARC->栈block->弱指针运行结果](https://sylarimage.oss-cn-shenzhen.aliyuncs.com/uPic/03d9b8c0de7e468a8ff301efdac840cf~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

运行结果显示栈上的**block**使用弱指针`__weak CLPerson *person`，没有影响`person`所指向对象的生命周期，出了临时作用域的之后就被释放了。







## Block类型

Block有3种类型

![Block的类型](https://sylarimage.oss-cn-shenzhen.aliyuncs.com/uPic/945af9e471d14e7d988006092c49ca69~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)



回顾一下程序的内存布局

> - **代码段** 占用空间很小，一般存放在内存的低地址空间，我们平时编写的所有代码，就是放在这个区域
> - **数据段** 用来存放全局变量
> - **堆区** 是动态分配内存的，用来存放我们代码中通过alloc生成的对象，动态分配内存的特点是需要程序员申请内存和管理内存。例如OC中alloc生成的对象需要调用releas方法释放【MRC下】，C中通过malloc生成的对象必须要通过free()去释放。
> - **栈区** 系统自动分配和销毁内存，用于存放函数内生成的局部变量



![block的存放区域](https://sylarimage.oss-cn-shenzhen.aliyuncs.com/uPic/3c59a6d65fae44718ad27bf973c2b413~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)



### (1) **NSGlobalBlock**(也就是_NSConcreteGlobalBlock)

> 如果一个block内部没有使用/访问 ***自动变量(auto变量)***，那么它的类型即为`__NSGlobalBlock__`，它会被存储在应用程序的 **数据段**



### (2) **NSStaticBlock**(也就是_NSConcreteStaticBlock)

> 如果一个block有使用/访问 ***自动变量(auto变量)***，那么它的类型即为`__NSStaticBlock__`，它会被存储在应用程序的 **栈区**



### (3) **NSMallocBlock**(也就是_NSConcreteMallocBlock)

> 对`__NSMallocBlock__`调用`copy`方法，就可以转变成`__NSMallocBlock__`，它会被存储在堆区上



### 总结

![block的类型总结](https://sylarimage.oss-cn-shenzhen.aliyuncs.com/uPic/9a944ca31af943a6bb7e836c3e42d3e5~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) 



对每一种类型的block调用copy后的结果如下

![img](https://sylarimage.oss-cn-shenzhen.aliyuncs.com/uPic/ec12027162624975bbc16054207bcbfc~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)



**在ARC环境下，编译器会根据情况自动将栈上的block复制到堆上，例如以下的情况**

- block作为函数参数返回的时候
- 将block赋值给`__strong`指针的时候
- block作为**Cocoa API**中方法名里面含有`usingBlock`的方法参数时
- block作为**GCD API**的方法参数的时候





##  Block生命周期

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





### weak的实现原理

在原对象释放之后，weak对象就会变成null，防止野指针。所以就输出了null了。

那么我们怎么才能在weakSelf之后，block里面还能继续使用weakSelf之后的对象呢？

究其根本原因就是weakSelf之后，无法控制什么时候会被释放，为了保证在block内不会被释放，需要添加_strong。

在block里面使用的_strong修饰的weakSelf是为了在函数生命周期中防止self提前释放。strongSelf是一个自动变量当block执行完毕就会释放自动变量strongSelf不会对self进行一直进行强引用。







## __block 修饰符



### __block修饰符原理：

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



### 

当__block修饰外界变量时

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



将代码编译成C++源码



![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-02-14-084559.jpg)



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



### 如何从栈指向堆，并建立联系呢？

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

通过当前栈空间主结构体上的`__Block_byref_a_0`结构体指针，访问指向堆空间的`__forwarding`成员，**并获取堆空间上变量的值**。

**当然，不仅__block修饰的变量会这样，前文的对象类型变量同样会在copy函数内部被转化成类似的结构体进行处理。**



`__block`修饰的属性在底层会生成响应的结构体，保存原始变量的指针，并传递一个指针地址给block——因此是指针拷贝



`__block` 所起到的作用就是只要观察到该变量被 block 所持有，就将“外部变量”在栈中的内存地址放到了堆中。进而在block内部也可以修改外部变量的值。

__block修饰的变量成了对象![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-23-035859.jpg)







###  __forwarding存在意义

不论在任何内存位置,都可以顺利访问同一个__block变量.




![__block + 基本类型变量](https://sylarimage.oss-cn-shenzhen.aliyuncs.com/uPic/a028c8db45ed474e9d8304da0dc9cf46~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

![__block + 对象类型变量](https://sylarimage.oss-cn-shenzhen.aliyuncs.com/uPic/9cc99c644b574d338181ffeb35480230~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

![__block + __weak + 对象类型变量](https://sylarimage.oss-cn-shenzhen.aliyuncs.com/uPic/ca9f77a55ae7451eba6aacb2ff839440~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)





## Reference

[1 深入研究 Block 捕获外部变量和 __block 实现原理](https://halfrost.com/ios_block/)

[2 深入理解iOS的block](https://juejin.cn/post/6844903893176958983)

[3 iOS中__block 关键字的底层实现原理](https://www.jianshu.com/p/404ff9d3cd42)

[4 iOS探索 全方位解读Block](https://juejin.cn/post/6844904181778612231)

[5 iOS - block原理解读（三）](https://www.jianshu.com/p/9af777c7d222)

[探寻Block的本质（6）—— __block的深入分析__block的使用场景 大家应该都知道，如果想在block - 掘金 (juejin.cn)](https://juejin.cn/post/6968314665667395620)