#  Thread 多线程


## 线程与进程
**进程:** 在系统中正在运行的一个应用程序,每个进程之间都是独立的，每个进程均运行在其专用且受保护的内存空间内。进程是CPU分配资源和调度的单位。
**线程:** 一个进程(程序)的所有任务都在线程中执行，每个进程至少有一个线程(主线程)。线程是CPU调度（执行任务）的最小单位，其实质就是一段代码（一个任务）。

##  浅谈GCD
### 1. GCD的两个核心概念是:`任务`和`队列`. 

#### 任务与队列

* 任务 : 在block中执行的代码块 
* 队列 : 用来存放任务的
  
#### 队列 和 线程的区别: 
队列中存放的任务最后都要由线程来执行! 
队列的原则:先进先出,后进后出

#### 队列分类：
1.串行队列 2.并发队列 3.主队列 4.全局队列

* 串行队列:任务一个接一个的执行 
* 并发队列:队列中的任务并发执行 
* 主队列:跟主线程相关的队列,主队列里面的内容都会在主线程中执行 
* 全局队列:一个特殊的并发队列

#### 并发队列和全局队列的区别: 
* 并发队列有名称,可以跟踪错误.全局队列没有. 
* 在ARC中两个队列不需要考虑释放内存,但是在MRC中并发队列创造出来的需要 release 操作,而全局队列只有一个不需要. 
* 一般在开发过程中我们使用全局队列

### 2.同步和异步: 
#### 同步异步
同步:只能在当前线程中执行任务,不具备开启新线程的能力 
异步:可以在新的线程中执行任务,具备开启新线程的能力

同步执行任务: 
dispatch_sync(队列,任务) 
异步执行: 
dispatch_async(队列,任务)

#### 队列和执行方式组合的效果:
1) 串行队列同步执行，既在当前线程中顺序执行 
2) 串行队列异步执行，开辟一条新的线程，在该线程中顺序执行 
3) 并行队列同步执行，不开辟线程，在当前线程中顺序执行 
4) 并行队列异步执行，开辟多个新的线程，并且线程会重用，无序执行 
5) 主队列异步执行，不开辟新的线程，顺序执行 
6) 主队列同步执行，会造成死锁（’主线程’和’主队列’相互等待,卡住主线程）


## 死锁原因
队列引起的循环等待

## 同步/异步和串行/并发
* dispatch_sync(serial_queue,^{//任务});
* dispatch_async(serial_queue,^{//任务});
* dispatch_sync(concurrent_queue,^{//任务});
* dispatch_async(concurrent_queue,^{//任务});

## 同步串行
### 1.
```
-(void)viewDidLoad{
    dispatch_sync(dispatch_get_main_queue(),^{
        [self doSomething];
    });
//死锁
}
```

![](http://img.isylar.com/media/15498666666148.jpg)

在主队列上提交了 `viewDidLoad` 与 `GCD Block的任务`,无论任务中哪一个，最终都要提交到`主线程`中处理.先分派`viewDidLoad`到主线程，由于队列`FIFO`,`viewDidLoad`的调用结束又要等待`Block`的调用结束，`Block`又在等待`viewDidLoad`

**只要是同步方式提交任务，无论是提交到并发队列还是串行队列，最终都是在当前线程执行**
### 2.

```
-(void)viewDidLoad{
    dispatch_sync(serialQueue,^{
        [self doSomething];
    });
//没问题
}
```

![](http://img.isylar.com/media/15498684664311.jpg)

`viewDidLoad`添加到主队列上,提交到`主线程`上执行.`viewDidLoad`执行到某个时段时候，同步提交一个任务到一个`串行队列`上面,由于是`同步提交`任务，意味着要在当前线程执行，所以`串行队列`提交的任务也是在`主线程`上面执行,`串行队列`任务在主线程上执行完之后，再继续执行`viewDidLoad`后面的任务

## 3.

```
-(void)viewDidLoad{
    NSLog(@"1");
    dispatch_sync(global_queue,^{
        NSLog(@"2");
        dispatch_sync(global_queue,^{
            NSLog(@"3");
        });
        NSLog(@"4");
    });
    NSLog(@"5");
}
```

`12345`

## 异步串行
##1.

```
-(void)viewDidLoad{
    dispatch_async(dispatch_get_main_queue(),^{
        [self doSomething];
    });
}
```

## 异步并发
##1.

```
dispatch_async(global_queue,^{
    NSLog(@"1");
    [self performSelector:@selector(printLog) withObject:nil afterDelay0];
    NSLog(@"3");
});

-(void)printLog{NSLog(@"2");}
```

`13`,因为子线程不会主动创建`runloop`，`performSelector:withObject:afterDelay`,即使延时0s，也是要创建相应添加到`runloop`逻辑,如果没有`runloop`是不会添加到上面，所以不会触发.

## dispatch_barrier_async()
###怎么利用GCD实现多读单写？
* 读者读者并发
* 读者写者互斥
* 写者写者互斥

### 多读单写处理
![](http://img.isylar.com/media/15498704537435.jpg)

### 多读单写方案

```
dispatch_barrier_async(concurrent_queue,^{//写操作});
```

```
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


4. GCD优点：GCD主要与block结合使用。


**引申:**

**使用NSOperation和NSOperationQueue的优点：**

1. 可以取消操作：在运行任务前，可以在NSOperation对象调用cancel方法，标明此任务不需要执行。但是GCD队列是无法取消的，因为它遵循“安排好之后就不管了（fire and forget）”的原则。
2. 可以指定操作间的依赖关系：例如从服务器下载并处理文件的动作可以用操作来表示。而在处理其他文件之前必须先下载“清单文件”。而后续的下载工作，都要依赖于先下载的清单文件这一操作。
3. 监控NSOperation对象的属性：可以通过KVO来监听NSOperation的属性：可以通过isCancelled属性来判断任务是否已取消；通过isFinished属性来判断任务是否已经完成。
4. 可以指定操作的优先级：操作的优先级表示此操作与队列中其他操作之间的优先关系，我们可以指定它

### 状态控制

* 如果只重写`main`方法,底层控制变更任务执行完成状态，以及任务退出
* 如果重写了`start`方法，自行控制状态(什么时候是`isExecuting`,`isFinish`状态等等)

**系统怎么移除一个 `isFinished==YES` 的NSOperation的**
通过`KVO`