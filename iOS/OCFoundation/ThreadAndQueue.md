# 多线程



## 进程，线程

进程：资源分配的最小单位。在iOS中一个应用的启动就是开启了一个进程。每个`进程`之间是独立的，每个`进程`均运行在专用的且受保护的内存

线程：CPU调度的最小单位。一个进程里会有多个线程。`进程`想要执行任务，必须得有`线程`，`进程`至少要有一条`线程`



<img src="http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-02-26-064756.jpg" style="zoom:50%;" />



## 队列（queue）

 队列（queue）：是先进先出（`FIFO： First-In-First-Out`）的线性表，在具体应用中通常用链表或者数组来实现。队列的操作方式和堆栈类似，唯一的区别在于队列只允许新数据在后端进行添加。

![img](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-02-26-065337.jpg)

### 队列和线程的关系

两者是没有关系的，可以这么理解：

- `队列`负责调度任务，`线程`执行任务
- 在银行（`进程`）中，有4个工作窗口（`线程`），而只有一条队伍（`队列`）
- 窗口（`线程`）只负责为排队的人办理业务，并不会管队伍（`队列`）是怎么排的



### 串行队列和并发队列

`并发队列（Concurrent Queue`），不是`并行队列`！！！

什么是`串行队列`和`并发队列` 呢？`串行队列`和`并发队列`中的`串行`和`并发`是`队列`的定语，可以加个`的`，`串行的队列`和`并发的队列`；所以`串行队列`和`并行队列`说到底还是`队列`，既然是`队列`，肯定是要先进先出（FIFO, First-In-First-Out）的，记住这一点很重要。

`串行队列`：说明这个队列中的任务要`串行`执行，也就是一个一个的执行，必须等上一个任务执行完成之后才能开始下一个，而且一定是按照先进先出的顺序执行的，比如`串行队列`里面有4个任务，进入队列的顺序是a、b、c、d，那么一定是先执行a，并且等任务a完成之后，再执行b… 。多任务中某时刻只能有一个任务被运行；

`并发队列`：说明这个队列中的任务可以`并发`执行，也就任务可以同时执行,比如`并发队列`里面有4个任务，进入队列的顺序是a、b、c、d，那么一定是先执行a，再执行b…，但是执行b的时候a不一定执行完成，而且a和b具体哪个先执行完成是不确定的， 具体同时执行几个，由系统控制(GCD中不能直接设置并发数，可以通过创建信号量的方式实现，NSOperationQueue可以直接设置)，但是肯定也是按照先进先出（FIFO, First-In-First-Out）的原则调用的。



我们常用的调度队列有以下几种：

```shell
// 串行队列
dispatch_queue_t serialQueue = dispatch_queue_create("com.gcd.serialQueue", DISPATCH_QUEUE_SERIAL);
// 并发队列
dispatch_queue_t concurrentQueue = dispatch_queue_create("com.gcd.concurrentQueue", DISPATCH_QUEUE_CONCURRENT);
// 全局并发队列
dispatch_queue_t globalQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
// 主队列
let mainQueue = DispatchQueue.main
```



### `并发`和`并行`

并行的英文是parallelism，并发的英文时concurrency ，

1. 并发表示逻辑概念上的同时，并行表示物理概念上的同时。
2. 并发指的是代码的性质，并行指的是物理运行状态
3. 并发是说进程B的开始时间是在进程A的开始时间与结束时间之间，我们就说A和B是并发的。并行指同一时间两个线程运行在不同的cpu。
4. 并发是同时处理很多事情（dealing with lots of things at once），并行是同时执行很多事情（doing lots of things at once）；
5. 并发可认为是一种逻辑结构的设计模式。你可以用并发的设计方式去编写程序，然后运行在一个单核cpu上，通过cpu动态地逻辑切换制造出并行的假象。此时，你的程序不是并行，但是是并发的。如果将并发的程序运行在多核CPU上，此时你的程序可以认为是并行。并行更关注的是程序的执行（execution）；
6. 对于单核CPU来说，并行至少两个CPU才行；而并发一个cpu也可以，两个任务交替执行即可；

综上所述：并发更多的是编写程序上的概念，并行是物理CPU执行上的概念。并发可以用并行的方式实现。并发是从编程的角度来解释的，并行是从cpu执行任务的角度来看的，一般来说我们只能编写并发的程序，却无法保证编写出并行的程序。

可以把并发和并行当成不同维度的东西。并发是从程序员编写程序的角度来看的。并行是从程序的物理执行上来看的。



如何介绍并发和并行的区别:



![img](https://dandan2009.github.io/img/15248975776264.jpg)

### 同步和异步

GCD中的`同步`和`异步`是针对任务的执行来说的，也就是同步执行任务和异步执行任务。 同步或异步描述的是task与其上下文之间的关系

同步执行：可以理解为，调用函数时(或执行一个代码块时)，必须等这个函数（或代码块）执行完成之后才会执行下面的代码。 同步执行 一般在当前线程中执行任务，不会开启新的线程。

异步：不管调用的函数有没有执行完，都会继续执行下面的代码。具备开启新线程的能力。

同步和异步的主要区别是向队列里面添加任务时是立即返回还是等添加的任务完成之后再返回。

```
dispatch_sync(dispatch_queue_t queue, dispatch_block_t block);
dispatch_async(dispatch_queue_t queue, dispatch_block_t block);
```

dispatch_sync就是添加同步任务的，添加任务的时候，必须等block里面的代码执行完，dispatch_sync这个函数才能返回。

dispatch_async是添加异步任务的，添加任务的时候会立即返回，不管block里面的代码是否执行。



### 串行、并发和同步、异步相互结合能否开启新线程

|            | 串行队列     | 并发队列     | 主队列       |
| ---------- | ------------ | ------------ | ------------ |
| 同步(sync) | 不开启新线程 | 不开启新线程 | 不开启新线程 |
| 异步       | 开启新线程   | 开启新线程   | 不开启新线程 |



## 样例

### 串行队列异步任务

```objective-c
dispatch_queue_t serialQueue = dispatch_queue_create("Dan-serial", DISPATCH_QUEUE_SERIAL);
    for(int i = 0; i < 5; i++){
        dispatch_async(serialQueue, ^{
            NSLog(@"我开始了：%@ , %@",@(i),[NSThread currentThread]);
            [NSThread sleepForTimeInterval: i % 3];
        });
    }
输出如下：    
我开始了：0 , <NSThread: 0x60400027e480>{number = 3, name = (null)}
我开始了：1 , <NSThread: 0x60400027e480>{number = 3, name = (null)}
我开始了：2 , <NSThread: 0x60400027e480>{number = 3, name = (null)}
我开始了：3 , <NSThread: 0x60400027e480>{number = 3, name = (null)}
我开始了：4 , <NSThread: 0x60400027e480>{number = 3, name = (null)}
```

按顺输出的，是在同一个线程，而且开启了新线程.



### 串行队列同步任务

```objective-c
for(int i = 0; i < 5; i++){
        dispatch_sync(serialQueue, ^{
            NSLog(@"我开始了：%@ , %@",@(i),[NSThread currentThread]);
            [NSThread sleepForTimeInterval: i % 3];
        });
    }  
输出如下：    
我开始了：0 , <NSThread: 0x60000006d8c0>{number = 1, name = main}
我开始了：1 , <NSThread: 0x60000006d8c0>{number = 1, name = main}
我开始了：2 , <NSThread: 0x60000006d8c0>{number = 1, name = main}
我开始了：3 , <NSThread: 0x60000006d8c0>{number = 1, name = main}
我开始了：4 , <NSThread: 0x60000006d8c0>{number = 1, name = main}
```

按顺输出的，是在同一个线程，但是没有开启新线程，是在主线程执行的



### 并发队列异步任务

```objective-c
dispatch_queue_t concurrent_queue = dispatch_queue_create("DanCONCURRENT", DISPATCH_QUEUE_CONCURRENT);
    for(int i = 0; i < 5; i++){
        dispatch_async(concurrent_queue, ^{
            NSLog(@"我开始了：%@ , %@",@(i),[NSThread currentThread]);
            [NSThread sleepForTimeInterval: i % 3];
            NSLog(@"执行完成：%@ , %@",@(i),[NSThread currentThread]);
        });
    }
输出如下：
我开始了：0 , <NSThread: 0x600000462340>{number = 3, name = (null)}
我开始了：2 , <NSThread: 0x604000269380>{number = 6, name = (null)}
我开始了：3 , <NSThread: 0x604000269180>{number = 5, name = (null)}
我开始了：1 , <NSThread: 0x600000461d80>{number = 4, name = (null)}
执行完成：0 , <NSThread: 0x600000462340>{number = 3, name = (null)}
执行完成：3 , <NSThread: 0x604000269180>{number = 5, name = (null)}
我开始了：4 , <NSThread: 0x600000462340>{number = 3, name = (null)}
执行完成：1 , <NSThread: 0x600000461d80>{number = 4, name = (null)}
执行完成：4 , <NSThread: 0x600000462340>{number = 3, name = (null)}
执行完成：2 , <NSThread: 0x604000269380>{number = 6, name = (null)}

```

并发执行的，而且开启了不止一个新线程。



### 并发队列同步任务

```objective-c
dispatch_queue_t concurrent_queue = dispatch_queue_create("DanCONCURRENT", DISPATCH_QUEUE_CONCURRENT);
for(int i = 0; i < 5; i++){
        dispatch_sync(concurrent_queue, ^{
            NSLog(@"我开始了：%@ , %@",@(i),[NSThread currentThread]);
            [NSThread sleepForTimeInterval: i % 3];
            NSLog(@"执行完成：%@ , %@",@(i),[NSThread currentThread]);
        });
    }
    
输出如下：
我开始了：0 , <NSThread: 0x60000007ec80>{number = 1, name = main}
执行完成：0 , <NSThread: 0x60000007ec80>{number = 1, name = main}
我开始了：1 , <NSThread: 0x60000007ec80>{number = 1, name = main}
执行完成：1 , <NSThread: 0x60000007ec80>{number = 1, name = main}
我开始了：2 , <NSThread: 0x60000007ec80>{number = 1, name = main}
执行完成：2 , <NSThread: 0x60000007ec80>{number = 1, name = main}
我开始了：3 , <NSThread: 0x60000007ec80>{number = 1, name = main}
执行完成：3 , <NSThread: 0x60000007ec80>{number = 1, name = main}
我开始了：4 , <NSThread: 0x60000007ec80>{number = 1, name = main}
执行完成：4 , <NSThread: 0x60000007ec80>{number = 1, name = main}
```

程序没有并发执行，而且没有开启新线程，是在主线程执行的



 >使用dispatch_sync 添加同步任务，必须等添加的block执行完成之后才返回。 既然要执行block，肯定需要线程，要么新开线程执行，要么再已存在的线程（包括当前线程）执行。   dispatch_sync的官方注释里面有这么一句话：
 >
 > As an optimization, dispatch_sync() invokes the block on the current thread when possible. 
 >
 >作为优化，**如果可能，直接在当前线程调用这个block**。所以，一般，在大多数情况下，通过dispatch_sync添加的任务，在哪个线程添加就会在哪个线程执行。 上面我们添加的任务的代码是在主线程，所以就直接在主线程执行了。



### 串行队列里的任务都在一个线程上执行？

测试如下

```objective-c
 - (void)viewDidLoad {
      dispatch_queue_t serialQueue = dispatch_queue_create("Dan-serial", DISPATCH_QUEUE_SERIAL);

    dispatch_sync(serialQueue, ^{
        // block 1
        NSLog(@"current 1: %@", [NSThread currentThread]);
    });

    dispatch_sync(serialQueue, ^{
        // block 2
        NSLog(@"current 2: %@", [NSThread currentThread]);
    });

    dispatch_async(serialQueue, ^{
        // block 3
        NSLog(@"current 3: %@", [NSThread currentThread]);
    });

    dispatch_async(serialQueue, ^{
        // block 4
        NSLog(@"current 4: %@", [NSThread currentThread]);
    });
}
    //结果如下
    //    current 1: <NSThread: 0x600000071600>{number = 1, name = main}
    //    current 2: <NSThread: 0x600000071600>{number = 1, name = main}
    //    current 3: <NSThread: 0x60400027bcc0>{number = 3, name = (null)}
    //    current 4: <NSThread: 0x60400027bcc0>{number = 3, name = (null)}
```

可以看到，向串行队列添加的同步任务在主线程执行的，和上面的结论一致(通过dispatch_sync添加的任务，在哪个线程添加就会在哪个线程执行)。 异步任务在新开的线程执行的，而且只开了一个线程



再做如下测试：

```objective-c
  - (void)viewDidLoad {
      dispatch_queue_t queue = dispatch_queue_create("Dan", NULL);
       dispatch_async(queue, ^{
        NSLog(@"current : %@", [NSThread currentThread]);
        dispatch_queue_t serialQueue = dispatch_queue_create("Dan-serial", DISPATCH_QUEUE_SERIAL);

        dispatch_sync(serialQueue, ^{
            // block 1
            NSLog(@"current 1: %@", [NSThread currentThread]);
        });

        dispatch_sync(serialQueue, ^{
            // block 2
            NSLog(@"current 2: %@", [NSThread currentThread]);
        });

        dispatch_async(serialQueue, ^{
            // block 3
            NSLog(@"current 3: %@", [NSThread currentThread]);
        });

        dispatch_async(serialQueue, ^{
            // block 4
            NSLog(@"current 4: %@", [NSThread currentThread]);
        });
    });
  }

// 结果如下
//    current  : <NSThread: 0x604000263440>{number = 3, name = (null)}
//    current 1: <NSThread: 0x604000263440>{number = 3, name = (null)}
//    current 2: <NSThread: 0x604000263440>{number = 3, name = (null)}
//    current 3: <NSThread: 0x604000263440>{number = 3, name = (null)}
//    current 4: <NSThread: 0x604000263440>{number = 3, name = (null)}
```



可以看到：

- 在主线程向自定义的串行队列添加的同步任务，直接在主线程执行
- 在主线程向自定义的串行队列添加的异步任务，会开一个新线程
- 在非主线程向自定义的串行队列添加的同步任务，直接在当期线程执行
- 在非主线程向自定义的串行队列添加的异步任务，直接在当期线程执行

结论：使用dispatch_sync函数添加到serial dispatch queue中的任务，其运行的task往往与所在的上下文是同一个thread；使用dispatch_async函数添加到serial dispatch queue中的任务，一般会(不一定)新开一个线程，但是不同的异步任务用的是同一个线程。



## 死锁

队列引起的循环等待

有以下代码

```objective-c
-(void)viewDidLoad{
    NSLog(@"执行任务1");
    dispatch_queue_t queue = dispatch_get_main_queue();
    dispatch_sync(queue, ^{
        NSLog(@"执行任务2");
    });

    NSLog(@"执行任务3");
		//死锁
}
```

这段代码会输出`执行任务1`，然后发生死锁，导致crash。



![img](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-24-143523.jpg)

在主队列上提交了 `viewDidLoad` 与 `GCD Block的任务`,无论任务中哪一个，最终都要提交到`主线程`中处理.先分派`viewDidLoad`到主线程，由于队列`FIFO`,`viewDidLoad`的调用结束又要等待`Block`的调用结束，`Block`又在等待`viewDidLoad`。(ViewDidLoad方法里面包含了Block方法，所以任务队列里面有ViewDidLoad+Block 的任务)



**只要是同步方式提交任务，无论是提交到并发队列还是串行队列，最终都是在当前线程执行**



有以下代码

```objective-c
-(void)viewDidLoad{
    dispatch_sync(serialQueue,^{
        [self doSomething];
    });
	//没问题
}
```

![img](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-24-143708.jpg)

`viewDidLoad`添加到主队列上,提交到`主线程`上执行.`viewDidLoad`执行到某个时段时候，同步提交一个任务到一个`串行队列`上面,由于是`同步提交`任务，意味着要在当前线程执行，所以`串行队列`提交的任务也是在`主线程`上面执行,`串行队列`任务在主线程上执行完之后，再继续执行`viewDidLoad`后面的任务



## Runloop与子线程

有以下代码

```objective-c
dispatch_async(global_queue,^{
    NSLog(@"1");
    [self performSelector:@selector(printLog) withObject:nil afterDelay0];
    NSLog(@"3");
});

-(void)printLog{
    NSLog(@"2");
}
//13
```

因为子线程不会主动创建`runloop`，`performSelector:withObject:afterDelay`,即使延时0s，也是要创建相应添加到`runloop`逻辑,如果没有`runloop`是不会添加到上面，所以不会触发.(创建runloop后也需要Run)



## dispatch_barrier_sync & dispatch_barrier_async

应用场景：`同步锁`

![img](https://sylarimage.oss-cn-shenzhen.aliyuncs.com/uPic/17169e7060708675~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.png)



`并发执行异步队列`会开辟线程，而任务也会因为任务复杂度和cpu的调度导致各个乱序执行完毕，比如上图中的`任务3`明明是先于`任务4`执行，但是晚于`任务4`执行完毕

此时GCD就提供了两个API——`dispatch_barrier_sync`和`dispatch_barrier_async`，使用这两个API就能将多个任务进行分组——等栅栏前**追加到队列中**的任务执行完毕后，再将栅栏后的任务追加到队列中。简而言之，就是先执行`栅栏前任务`，再执行`栅栏任务`，最后执行`栅栏后任务`

#### 1.串行队列使用栅栏函数

```objective-c
- (void)test {
    dispatch_queue_t queue = dispatch_queue_create("Felix", DISPATCH_QUEUE_SERIAL);

    NSLog(@"开始——%@", [NSThread currentThread]);
    dispatch_async(queue, ^{
        sleep(2);
        NSLog(@"延迟2s的任务1——%@", [NSThread currentThread]);
    });
    NSLog(@"第一次结束——%@", [NSThread currentThread]);

//    dispatch_barrier_async(queue, ^{
//        NSLog(@"----------栅栏任务----------%@", [NSThread currentThread]);
//    });
//    NSLog(@"栅栏结束——%@", [NSThread currentThread]);

    dispatch_async(queue, ^{
        sleep(1);
        NSLog(@"延迟1s的任务2——%@", [NSThread currentThread]);
    });
    NSLog(@"第二次结束——%@", [NSThread currentThread]);
}
```

不使用栅栏函数

```objective-c
开始——<NSThread: 0x600001068900>{number = 1, name = main}
第一次结束——<NSThread: 0x600001068900>{number = 1, name = main}
第二次结束——<NSThread: 0x600001068900>{number = 1, name = main}
延迟2s的任务1——<NSThread: 0x600001025ec0>{number = 3, name = (null)}
延迟1s的任务2——<NSThread: 0x600001025ec0>{number = 3, name = (null)}
```

使用栅栏函数

```objective-c
开始——<NSThread: 0x6000001bcf00>{number = 1, name = main}
第一次结束——<NSThread: 0x6000001bcf00>{number = 1, name = main}
栅栏结束——<NSThread: 0x6000001bcf00>{number = 1, name = main}
第二次结束——<NSThread: 0x6000001bcf00>{number = 1, name = main}
延迟2s的任务1——<NSThread: 0x6000001fcf00>{number = 5, name = (null)}
----------栅栏任务----------<NSThread: 0x6000001bcf00>{number = 1, name = main}
延迟1s的任务2——<NSThread: 0x6000001fcf00>{number = 5, name = (null)}
```

栅栏函数的作用是将队列中的任务进行分组，所以我们只要关注`任务1`、`任务2`

![img](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-03-01-145813.jpg)

**结论：由于`串行队列异步执行`任务是一个接一个执行完毕的，所以使用栅栏函数没意义**



#### 2.并发队列使用栅栏函数

```objective-c
- (void)test {
    dispatch_queue_t queue = dispatch_queue_create("Felix", DISPATCH_QUEUE_CONCURRENT);

    NSLog(@"开始——%@", [NSThread currentThread]);
    dispatch_async(queue, ^{
        sleep(2);
        NSLog(@"延迟2s的任务1——%@", [NSThread currentThread]);
    });
    NSLog(@"第一次结束——%@", [NSThread currentThread]);

//    dispatch_barrier_async(queue, ^{
//        NSLog(@"----------栅栏任务----------%@", [NSThread currentThread]);
//    });
//    NSLog(@"栅栏结束——%@", [NSThread currentThread]);

    dispatch_async(queue, ^{
        sleep(1);
        NSLog(@"延迟1s的任务2——%@", [NSThread currentThread]);
    });
    NSLog(@"第二次结束——%@", [NSThread currentThread]);
}

```

不使用栅栏函数

```
开始——<NSThread: 0x600002384f00>{number = 1, name = main}
第一次结束——<NSThread: 0x600002384f00>{number = 1, name = main}
第二次结束——<NSThread: 0x600002384f00>{number = 1, name = main}
延迟1s的任务2——<NSThread: 0x6000023ec300>{number = 5, name = (null)}
延迟2s的任务1——<NSThread: 0x60000238c180>{number = 7, name = (null)}
```

使用栅栏函数

```
开始——<NSThread: 0x600000820bc0>{number = 1, name = main}
第一次结束——<NSThread: 0x600000820bc0>{number = 1, name = main}
栅栏结束——<NSThread: 0x600000820bc0>{number = 1, name = main}
第二次结束——<NSThread: 0x600000820bc0>{number = 1, name = main}
延迟2s的任务1——<NSThread: 0x600000863c80>{number = 4, name = (null)}
----------栅栏任务----------<NSThread: 0x600000863c80>{number = 4, name = (null)}
延迟1s的任务2——<NSThread: 0x600000863c80>{number = 4, name = (null)}
```



![img](https://sylarimage.oss-cn-shenzhen.aliyuncs.com/uPic/1716bf701e5fee4d~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.png)



**结论：由于`并发队列异步执行`任务是乱序执行完毕的，所以使用栅栏函数可以很好的控制队列内任务执行的顺序**



#### 3.dispatch_barrier_sync/dispatch_barrier_async区别

- `dispatch_barrier_async`：前面的任务执行完毕才会来到这里
- `dispatch_barrier_sync`：作用相同，但是这个会堵塞线程，影响后面的任务执行

将案例二中的`dispatch_barrier_async`改成`dispatch_barrier_sync`

```
开始——<NSThread: 0x600001040d40>{number = 1, name = main}
第一次结束——<NSThread: 0x600001040d40>{number = 1, name = main}
延迟2s的任务1——<NSThread: 0x60000100ce40>{number = 6, name = (null)}
----------栅栏任务----------<NSThread: 0x600001040d40>{number = 1, name = main}
栅栏结束——<NSThread: 0x600001040d40>{number = 1, name = main}
第二次结束——<NSThread: 0x600001040d40>{number = 1, name = main}
延迟1s的任务2——<NSThread: 0x60000100ce40>{number = 6, name = (null)}
```

**结论：dispatch_barrier_async可以控制队列中任务的执行顺序，而dispatch_barrier_sync不仅阻塞了队列的执行，也阻塞了线程的执行（尽量少用）**



#### 4.栅栏函数注意点

1. 尽量使用自定义的并发队列：
   - 使用`全局队列`起不到`栅栏函数`的作用
   - 使用`全局队列`时由于对全局队列造成堵塞，可能致使系统其他调用全局队列的地方也堵塞从而导致崩溃（并不是只有你在使用这个队列）
2. `栅栏函数只能控制同一并发队列`：打个比方，平时在使用AFNetworking做网络请求时为什么不能用栅栏函数起到同步锁堵塞的效果，因为AFNetworking内部有自己的队列



## dispatch_semaphore_t

应用场景：同步当锁, 控制GCD最大并发数

- `dispatch_semaphore_create()`：创建信号量
- `dispatch_semaphore_wait()`：等待信号量，信号量减1。当信号量`< 0`时会阻塞当前线程，根据传入的等待时间决定接下来的操作——如果**永久等待**将等到`信号（signal）`才执行下去
- `dispatch_semaphore_signal()`：释放信号量，信号量加1。当信号量`>= 0` 会执行wait之后的代码

下面这段代码要求使用信号量来按序输出（当然栅栏函数可以满足要求）

```objective-c
- (void)test {
    dispatch_queue_t queue = dispatch_queue_create("Felix", DISPATCH_QUEUE_CONCURRENT);

    for (int i = 0; i < 10; i++) {
        dispatch_async(queue, ^{
            NSLog(@"当前%d----线程%@", i, [NSThread currentThread]);
        });
        // 使用栅栏函数
        // dispatch_barrier_async(queue, ^{});
    }
}
```

利用信号量的API来进行代码改写

```objective-c
- (void)test {
    // 创建信号量
    dispatch_semaphore_t sem = dispatch_semaphore_create(0);
    dispatch_queue_t queue = dispatch_queue_create("Felix", DISPATCH_QUEUE_CONCURRENT);

    for (int i = 0; i < 10; i++) {
        dispatch_async(queue, ^{
            NSLog(@"当前%d----线程%@", i, [NSThread currentThread]);
            // 打印任务结束后信号量解锁
            dispatch_semaphore_signal(sem);
        });
        // 由于异步执行，打印任务会较慢，所以这里信号量加锁
        dispatch_semaphore_wait(sem, DISPATCH_TIME_FOREVER);
    }
}
```

输出结果

```
当前0----线程<NSThread: 0x600000c2c000>{number = 5, name = (null)}
当前1----线程<NSThread: 0x600000c2c000>{number = 5, name = (null)}
当前2----线程<NSThread: 0x600000c2c000>{number = 5, name = (null)}
当前3----线程<NSThread: 0x600000c2c000>{number = 5, name = (null)}
当前4----线程<NSThread: 0x600000c2c000>{number = 5, name = (null)}
当前5----线程<NSThread: 0x600000c2c000>{number = 5, name = (null)}
当前6----线程<NSThread: 0x600000c2c000>{number = 5, name = (null)}
当前7----线程<NSThread: 0x600000c2c000>{number = 5, name = (null)}
当前8----线程<NSThread: 0x600000c2c000>{number = 5, name = (null)}
当前9----线程<NSThread: 0x600000c2c000>{number = 5, name = (null)}
```

如果当创建信号量时传入值为1又会怎么样呢？

- `i=0`时有可能先`打印`，也可能会先发出`wait`信号量-1，但是`wait`之后信号量为0不会阻塞线程，所以进入`i=1`
- `i=1`时有可能先`打印`，也可能会先发出`wait`信号量-1，但是`wait`之后信号量为-1阻塞线程，等待`signal`再执行下去

```
当前1----线程<NSThread: 0x600001448d40>{number = 3, name = (null)}
当前0----线程<NSThread: 0x60000140c240>{number = 6, name = (null)}
当前2----线程<NSThread: 0x600001448d40>{number = 3, name = (null)}
当前3----线程<NSThread: 0x60000140c240>{number = 6, name = (null)}
当前4----线程<NSThread: 0x60000140c240>{number = 6, name = (null)}
当前5----线程<NSThread: 0x600001448d40>{number = 3, name = (null)}
当前6----线程<NSThread: 0x600001448d40>{number = 3, name = (null)}
当前7----线程<NSThread: 0x60000140c240>{number = 6, name = (null)}
当前8----线程<NSThread: 0x600001448d40>{number = 3, name = (null)}
当前9----线程<NSThread: 0x60000140c240>{number = 6, name = (null)}
```

结论：

- 创建信号量时传入值为1时，可以通过两次才堵塞
- 传入值为2时，可以通过三次才堵塞

## dispatch_source

应用场景：`GCDTimer`

#### 1.定义及使用

`dispatch_source`是一种基本的数据类型，可以用来监听一些底层的系统事件

- `Timer Dispatch Source`：定时器事件源，用来生成周期性的通知或回调
- `Signal Dispatch Source`：监听信号事件源，当有UNIX信号发生时会通知
- `Descriptor Dispatch Source`：监听文件或socket事件源，当文件或socket数据发生变化时会通知
- `Process Dispatch Source`：监听进程事件源，与进程相关的事件通知
- `Mach port Dispatch Source`：监听Mach端口事件源
- `Custom Dispatch Source`：监听自定义事件源

主要使用的API：

- `dispatch_source_create`: 创建事件源
- `dispatch_source_set_event_handler`: 设置数据源回调
- `dispatch_source_merge_data`: 设置事件源数据
- `dispatch_source_get_data`： 获取事件源数据
- `dispatch_resume`: 继续
- `dispatch_suspend`: 挂起
- `dispatch_cancle`: 取消

#### 2.自定义定时器

在iOS开发中一般使用`NSTimer`来处理定时逻辑，但`NSTimer`是依赖`Runloop`的，而`Runloop`可以运行在不同的模式下。如果`NSTimer`添加在一种模式下，当Runloop运行在其他模式下的时候，定时器就挂机了；又如果`Runloop`在阻塞状态，`NSTimer`触发时间就会推迟到下一个`Runloop`周期。因此`NSTimer`在计时上会有误差，并不是特别精确，而GCD定时器不依赖`Runloop`，计时精度要高很多

```
@property (nonatomic, strong) dispatch_source_t timer;
//1.创建队列
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
//2.创建timer
_timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, queue);
//3.设置timer首次执行时间，间隔，精确度
dispatch_source_set_timer(_timer, DISPATCH_TIME_NOW, 2.0 * NSEC_PER_SEC, 0.1 * NSEC_PER_SEC);
//4.设置timer事件回调
dispatch_source_set_event_handler(_timer, ^{
    NSLog(@"GCDTimer");
});
//5.默认是挂起状态，需要手动激活
dispatch_resume(_timer);
复制代码
```

使用`dispatch_source`自定义定时器注意点：

- `GCDTimer`需要`强持有`，否则出了作用域立即释放，也就没有了事件回调

- `GCDTimer`默认是挂起状态，需要手动激活

- `GCDTimer`没有`repeat`，需要封装来增加标志位控制

- `GCDTimer`如果存在循环引用，使用`weak+strong`或者提前调用`dispatch_source_cancel`取消timer

- `dispatch_resume`和`dispatch_suspend`调用次数需要平衡

- `source`在`挂起状态`下，如果直接设置`source = nil`或者重新创建`source`都会造成crash。正确的方式是在`激活状态`下调用`dispatch_source_cancel(source)`释放当前的`source`

  



## 为什么UI操作必须在主线程执行

首先UIKit不是线程安全的，多线程访问会导致UI效果不可预期，所以我们不能使用多个线程去处理UI。那既然要单线程处理UI为什么是在主线程呢，这是因为UIApplication作为程序的起点是在主线程初始化的，所以我们后续的UI操作也都要放到主线程处理



## 如何实现一个多读单写的功能？

- 读者读者并发
- 读者写者互斥
- 写者写者互斥



![img](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-24-143713.jpg)





```
	dispatch_barrier_async(concurrent_queue,^{
  	//写操作
  });
  
//同步读取指定数据
-(id)objectForKey:(NSString*)key{
    __block id obj;
    dispatch_sync(concurrent_queue,^{
        obj = xxxx;
    });
}

//写
-(void)setObject:(id)obj forKey:(NSString*)key{
    //异步栅栏调用设置数据
    dispatch_barrier_async(concurrent_queue,^{
        xxxxx;
    });
}
```



## NSOperation

### 1. NSOprationQueue 与 GCD 的区别与选用

1. GCD是底层的C语言构成的API，而NSOperationQueue及相关对象是Objc的对象。在GCD中，在队列中执行的是由block构成的任务，这是一个轻量级的数据结构；而Operation作为一个对象，为我们提供了更多的选择；
2. 在NSOperationQueue中，我们可以随时取消已经设定要准备执行的任务(当然，已经开始的任务就无法阻止了)，而GCD没法停止已经加入queue的block(其实是有的，但需要许多复杂的代码)；
3. GCD 只支持FIFO 的队列，而NSOperationQueue可以调整队列的执行顺序（通过调整权重）。NSOperationQueue可以方便的管理并发、NSOperation之间的优先级。

1. GCD优点：GCD主要与block结合使用。

**引申:**

**使用NSOperation和NSOperationQueue的优点：**

1. 可以取消操作：在运行任务前，可以在NSOperation对象调用cancel方法，标明此任务不需要执行。但是GCD队列是无法取消的，因为它遵循“安排好之后就不管了（fire and forget）”的原则。
2. 可以指定操作间的依赖关系：例如从服务器下载并处理文件的动作可以用操作来表示。而在处理其他文件之前必须先下载“清单文件”。而后续的下载工作，都要依赖于先下载的清单文件这一操作。
3. 监控NSOperation对象的属性：可以通过KVO来监听NSOperation的属性：可以通过isCancelled属性来判断任务是否已取消；通过isFinished属性来判断任务是否已经完成。
4. 可以指定操作的优先级：操作的优先级表示此操作与队列中其他操作之间的优先关系，我们可以指定它

### 状态控制

- 如果只重写`main`方法,底层控制变更任务执行完成状态，以及任务退出
- 如果重写了`start`方法，自行控制状态(什么时候是`isExecuting`,`isFinish`状态等等)

**系统怎么移除一个 `isFinished==YES` 的NSOperation的** 通过`KVO`





## Reference

[iOS 多线程--GCD 串行队列、并发队列以及同步执行、异步执行 - 蛋蛋的博客 | Dan Blog (dandan2009.github.io)](https://dandan2009.github.io/2018/04/15/multithreading-gcd/)

[iOS面试备战-多线程 | zhangferry的技术博客](https://zhangferry.com/2020/07/19/interview-multithreading/)

