# GCD Usage

GCD全称是`Grand Central Dispatch`，它是纯 C 语言，并且提供了非常多强大的函数

GCD的优势：

- GCD 是苹果公司为多核的并行运算提出的解决方案
- GCD 会自动利用更多的CPU内核（比如双核、四核）
- GCD 会自动管理线程的生命周期（创建线程、调度任务、销毁线程）
- 程序员只需要告诉 GCD 想要执行什么任务，不需要编写任何线程管理代码



## dispatch_sync & dispatch_async

多线程执行任务分为`dispatch_sync`同步执行任务和`dispatch_async`异步执行：

- ```
  dispatch_sync
  ```

  同步执行

  - 必须等待当前语句执行完毕，才会执行下一条语句
  - 不会开启线程
  - 在当前线程执行block的任务

- ```
  dispatch_async
  ```

  异步执行

  - 不用等待当前语句执行完毕，就可以执行下一条语句
  - 会开启线程执行block任务
  - 异步是多线程的代名词



## dispatch_queue_t



![img](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-02-28-041109.jpg)

多线程中队列分为`串行队列`(`Serial Dispatch Queue)和`并发队列(`Concurrent Dispatch Queue`)：



- 串行队列：线程执行只能依次逐一先后有序的执行，等待上一个执行完再执行下一个
  - 使用`dispatch_queue_create("xxx", DISPATCH_QUEUE_SERIAL)`创建串行队列
  - 亦可以使用`dispatch_queue_create("xxx", NULL)`创建串行队列（GCD底层会讲到）
- 主队列：绑定主线程，所有任务都在主线程中执行、经过特殊处理的串行的队列
  - 使用`dispatch_get_main_queue()`获取主队列
- 并发队列：线程可以同时一起执行，不需要等待上一个执行完就能执行下一个任务
  - 使用`dispatch_queue_create("xxx", DISPATCH_QUEUE_CONCURRENT);`创建并发队列
- 全局队列：系统提供的并发队列
  - 最简单的是使用`dispatch_get_global_queue(0, 0)`获取系统提供的并发队列
  - 第一个参数是优先级枚举值，默认优先级为`DISPATCH_QUEUE_PRIORITY_DEFAULT`=0
  - 优先级从高到低依次为`DISPATCH_QUEUE_PRIORITY_HIGH`、`DISPATCH_QUEUE_PRIORITY_DEFAULT`、`DISPATCH_QUEUE_PRIORITY_LOW`、`DISPATCH_QUEUE_PRIORITY_BACKGROUND`

## 串行/并发和同步/异步的排列组合



![img](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-03-01-142506.jpg)



> 主队列和全局队列单独考虑，组合结果以总结表格为准

#### 1.串行+同步

任务一个接一个执行，不开辟线程

```objective-c
- (void)test {
    NSLog(@"主线程-%@", [NSThread currentThread]);
    dispatch_queue_t queue = dispatch_queue_create("Felix", DISPATCH_QUEUE_SERIAL);
    for (int i = 0; i < 10; i++) {
        dispatch_sync(queue, ^{
            NSLog(@"串行&同步线程%d-%@", i, [NSThread currentThread]);
        });
    }
}
--------------------输出结果：-------------------
// 主线程-<NSThread: 0x600003b64fc0>{number = 1, name = main}
// 串行&同步线程0-<NSThread: 0x600003b64fc0>{number = 1, name = main}
// 串行&同步线程1-<NSThread: 0x600003b64fc0>{number = 1, name = main}
// ...按顺序输出
--------------------输出结果：-------------------
```

#### 2.串行+异步

任务一个接一个执行，会开辟线程

```objective-c
- (void)test {
    NSLog(@"主线程-%@", [NSThread currentThread]);
    dispatch_queue_t queue = dispatch_queue_create("Felix", DISPATCH_QUEUE_SERIAL);
    for (int i = 0; i < 10; i++) {
        dispatch_async(queue, ^{
            NSLog(@"串行&异步线程%d-%@", i, [NSThread currentThread]);
        });
    }
}
--------------------输出结果：-------------------
// 主线程-<NSThread: 0x600003b64fc0>{number = 1, name = main}
// 串行&异步线程0-<NSThread: 0x6000009b8880>{number = 6, name = (null)}
// 串行&异步线程1-<NSThread: 0x6000009b8880>{number = 6, name = (null)}
// ...按顺序输出
--------------------输出结果：-------------------
```

#### 3.并发+同步

任务一个接一个执行，不开辟线程

```objective-c
- (void)test {
    NSLog(@"主线程-%@", [NSThread currentThread]);
    dispatch_queue_t queue = dispatch_queue_create("Felix", DISPATCH_QUEUE_CONCURRENT);
    for (int i = 0; i < 10; i++) {
        dispatch_sync(queue, ^{
            NSLog(@"并发&同步线程%d-%@", i, [NSThread currentThread]);
        });
    }
}
--------------------输出结果：-------------------
// 主线程-<NSThread: 0x600003b64fc0>{number = 1, name = main}
// 串行&同步线程0-<NSThread: 0x600003b64fc0>{number = 1, name = main}
// 串行&同步线程1-<NSThread: 0x600003b64fc0>{number = 1, name = main}
// ...按顺序输出
--------------------输出结果：-------------------
```

#### 4.并发+异步

任务乱序执行，开辟线程

```objective-c
- (void)test {
    NSLog(@"主线程-%@", [NSThread currentThread]);
    dispatch_queue_t queue = dispatch_queue_create("Felix", DISPATCH_QUEUE_CONCURRENT);
    for (int i = 0; i < 10; i++) {
        dispatch_async(queue, ^{
            NSLog(@"并发&异步线程%d-%@", i, [NSThread currentThread]);
        });
    }
}
--------------------输出结果：-------------------
// 主线程-<NSThread: 0x600002a9cd40>{number = 1, name = main}
// 并发&异步线程1-<NSThread: 0x600002a9ca40>{number = 5, name = (null)}
// 并发&异步线程0-<NSThread: 0x600002add3c0>{number = 4, name = (null)}
// ...乱序输出
--------------------输出结果：-------------------
```

下面来看一下`主队列`和`全局队列`的使用情况：

#### 5.主队列+同步

相互等待，造成死锁

```objective-c
- (void)test {
    NSLog(@"主线程-%@", [NSThread currentThread]);
    dispatch_queue_t queue = dispatch_get_main_queue();
    for (int i = 0; i < 10; i++) {
        dispatch_sync(queue, ^{
            NSLog(@"主队列&同步线程%d-%@", i, [NSThread currentThread]);
        });
    }
}
--------------------输出结果：-------------------
// 主线程-<NSThread: 0x600001980d40>{number = 1, name = main}
// 崩溃...
--------------------输出结果：-------------------
```

#### 6.主队列+异步

任务一个接一个执行，不开辟线程

```objective-c
- (void)test {
    NSLog(@"主线程-%@", [NSThread currentThread]);
    dispatch_queue_t queue = dispatch_get_main_queue();
    for (int i = 0; i < 10; i++) {
        dispatch_async(queue, ^{
            NSLog(@"主队列&异步线程%d-%@", i, [NSThread currentThread]);
        });
    }
}
--------------------输出结果：-------------------
// 主线程-<NSThread: 0x600001980d40>{number = 1, name = main}
// 主队列&异步线程0-<NSThread: 0x600001980d40>{number = 1, name = main}
// 主队列&异步线程1-<NSThread: 0x600001980d40>{number = 1, name = main}
// ...按顺序输出
--------------------输出结果：-------------------
```

#### 7.全局队列+同步

任务一个接一个执行，不开辟线程（同并发+同步）

```objective-c
- (void)test {
    NSLog(@"主线程-%@", [NSThread currentThread]);
    dispatch_queue_t queue = dispatch_get_global_queue(0, 0);
    for (int i = 0; i < 10; i++) {
        dispatch_sync(queue, ^{
            NSLog(@"全局队列&同步线程%d-%@", i, [NSThread currentThread]);
        });
    }
}
--------------------输出结果：-------------------
// 主线程-<NSThread: 0x600001980d40>{number = 1, name = main}
// 全局队列&同步线程0-<NSThread: 0x60000099c080>{number = 1, name = main}
// 全局队列&同步线程1-<NSThread: 0x60000099c080>{number = 1, name = main}
// ...按顺序输出
--------------------输出结果：-------------------
```

#### 8.全局队列+异步

任务乱序执行，开辟线程（同并发+异步）

```objective-c
- (void)test {
    NSLog(@"主线程-%@", [NSThread currentThread]);
    dispatch_queue_t queue = dispatch_get_global_queue(0, 0);
    for (int i = 0; i < 10; i++) {
        dispatch_async(queue, ^{
            NSLog(@"全局队列&异步线程%d-%@", i, [NSThread currentThread]);
        });
    }
}
--------------------输出结果：-------------------
// 主线程-<NSThread: 0x600001cd4ec0>{number = 1, name = main}
// 全局队列&异步线程2-<NSThread: 0x600001c8eb00>{number = 3, name = (null)}
// 全局队列&异步线程3-<NSThread: 0x600001c82b80>{number = 7, name = (null)}
// ...乱序输出
--------------------输出结果：-------------------
```

**总结一下：**

| 执行\队列 | 串行队列             | 并发队列             | 主队列               | 全局队列             |
| --------- | -------------------- | -------------------- | -------------------- | -------------------- |
| 同步执行  | 按序执行，不开辟线程 | 按序执行，不开辟线程 | 死锁                 | 按序执行，不开辟线程 |
| 异步执行  | 按序执行，开辟线程   | 乱序执行，开辟线程   | 按序执行，不开辟线程 | 乱序执行，开辟线程   |





##dispatch_apply

`dispatch_apply`将指定的Block追加到指定的队列中重复执行，并等到全部的处理执行结束——相当于线程安全的for循环

应用场景：用来拉取网络数据后提前算出各个控件的大小，防止绘制时计算，提高表单滑动流畅性

- 添加到串行队列中——按序执行
- 添加到主队列中——死锁
- 添加到并发队列中——乱序执行
- 添加到全局队列中——乱序执行

```objective-c
- (void)test {
    /**
     param1：重复次数
     param2：追加的队列
     param3：执行任务
     */
    dispatch_queue_t queue = dispatch_queue_create("Felix", DISPATCH_QUEUE_SERIAL);
    NSLog(@"dispatch_apply前");
    dispatch_apply(10, queue, ^(size_t index) {
        NSLog(@"dispatch_apply的线程%zu-%@", index, [NSThread currentThread]);
    });
    NSLog(@"dispatch_apply后");
}
--------------------输出结果：-------------------
// dispatch_apply前
// dispatch_apply的线程0-<NSThread: 0x6000019f8d40>{number = 1, name = main}
// ...是否按序输出与串行队列还是并发队列有关
// dispatch_apply后
--------------------输出结果：-------------------
```



## dispatch_group_t

`dispatch_group_t`：调度组将任务分组执行，能监听任务组完成，并设置等待时间

应用场景：多个接口请求之后刷新页面

#### 1.dispatch_group_async

`dispatch_group_notify`在`dispatch_group_async`执行结束之后会受到通知

```objective-c
- (void)test {
    dispatch_group_t group = dispatch_group_create();
    dispatch_queue_t queue = dispatch_get_global_queue(0, 0);
    
    dispatch_group_async(group, queue, ^{
        NSLog(@"请求一完成");
    });
    
    dispatch_group_async(group, queue, ^{
        NSLog(@"请求二完成");
    });
    
    dispatch_group_notify(group, dispatch_get_main_queue(), ^{
        NSLog(@"刷新页面");
    });
}
--------------------输出结果：-------------------
// 请求二完成
// 请求一完成
// 刷新页面
--------------------输出结果：-------------------
```

#### 2.dispatch_group_enter & dispatch_group_leave

`dispatch_group_enter`和`dispatch_group_leave`成对出现，使进出组的逻辑更加清晰

```objective-c
- (void)test {
    dispatch_group_t group = dispatch_group_create();
    dispatch_queue_t queue = dispatch_get_global_queue(0, 0);
    
    dispatch_group_enter(group);
    dispatch_async(queue, ^{
        NSLog(@"请求一完成");
        dispatch_group_leave(group);
    });
    
    dispatch_group_enter(group);
    dispatch_async(queue, ^{
        NSLog(@"请求二完成");
        dispatch_group_leave(group);
    });
    
    dispatch_group_notify(group, dispatch_get_main_queue(), ^{
        NSLog(@"刷新页面");
    });
}
--------------------输出结果：-------------------
// 请求二完成
// 请求一完成
// 刷新页面
--------------------输出结果：-------------------
```

> 调度组要注意搭配使用，必须先进组再出组，缺一不可

#### 3.`dispatch_group_wait`使用

```
long dispatch_group_wait(dispatch_group_t group, dispatch_time_t timeout)
```

- group：需要等待的调度组

- timeout：等待的超时时间（即等多久）

  - 设置为`DISPATCH_TIME_NOW`意味着不等待直接判定调度组是否执行完毕
  - 设置为`DISPATCH_TIME_FOREVER`则会阻塞当前调度组，直到调度组执行完毕

- 返回值：为 `long`

  类型

  - 返回值为0——在指定时间内调度组完成了任务
  - 返回值不为0——在指定时间内调度组没有按时完成任务

将上述调度组代码进行改写

```objective-c
- (void)test {
    dispatch_group_t group = dispatch_group_create();
    dispatch_queue_t queue = dispatch_get_global_queue(0, 0);
    
    dispatch_group_enter(group);
    dispatch_async(queue, ^{
        NSLog(@"请求一完成");
        dispatch_group_leave(group);
    });
    
    dispatch_group_enter(group);
    dispatch_async(queue, ^{
        NSLog(@"请求二完成");
        dispatch_group_leave(group);
    });
    
    long timeout = dispatch_group_wait(group, DISPATCH_TIME_NOW);
//    long timeout = dispatch_group_wait(group, DISPATCH_TIME_FOREVER);
//    long timeout = dispatch_group_wait(group, dispatch_time(DISPATCH_TIME_NOW, 1 * NSEC_PER_SEC));
    NSLog(@"timeout=%ld", timeout);
    if (timeout == 0) {
        NSLog(@"按时完成任务");
    } else {
        NSLog(@"超时");
    }
    
    dispatch_group_notify(group, dispatch_get_main_queue(), ^{
        NSLog(@"刷新页面");
    });
}
--------------------输出结果：-------------------
// timeout=49
// 请求一完成
// 请求二完成
// 超时
// 刷新页面
--------------------输出结果：-------------------
```



## dispatch_barrier_sync & dispatch_barrier_async

应用场景：`同步锁`

![img](https://user-gold-cdn.xitu.io/2020/4/11/17169e7060708675?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

前文已经提过`并发执行异步队列`会开辟线程，而任务也会因为任务复杂度和cpu的调度导致各个乱序执行完毕，比如上图中的`任务3`明明是先于`任务4`执行，但是晚于`任务4`执行完毕



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

```
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
复制代码
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



![img](https://user-gold-cdn.xitu.io/2020/4/12/1716bf701e5fee4d?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

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

## GCD-API总结

| API                                              | 说明                                                         |
| ------------------------------------------------ | ------------------------------------------------------------ |
| dispatch_sync()                                  | 同步执行                                                     |
| dispatch_async()                                 | 异步执行                                                     |
| dispatch_queue_create()                          | 创建队列                                                     |
| dispatch_get_main_queue()                        | 获取主队列                                                   |
| dispatch_get_global_queue()                      | 获取全局队列                                                 |
| dispatch_after()                                 | 延时执行                                                     |
| dispatch_once()                                  | 一次性执行                                                   |
| dispatch_apply()                                 | 提交队列                                                     |
| dispatch_group_create()                          | 创建调度组                                                   |
| dispatch_group_async()                           | 执行进组任务                                                 |
| dispatch_group_enter()/   dispatch_group_leave() | 将调度组中的任务未执行完毕的任务数目加减1 (两个函数要配合使用) |
| dispatch_group_wait()                            | 设置等待时间(成功为0)                                        |
| dispatch_barrier_sync()                          | 同步栅栏函数                                                 |
| dispatch_barrier_async()                         | 异步栅栏函数                                                 |
| dispatch_group_notify()                          | 监听队列组执行完毕                                           |
| dispatch_semaphore_creat()                       | 创建信号量                                                   |
| dispatch_semaphore_wait()                        | 等待信号量                                                   |
| dispatch_semaphore_signal()                      | 释放信号量                                                   |
| dispatch_source_create                           | 创建源                                                       |
| dispatch_source_set_event_handler                | 设置源事件回调                                               |
| dispatch_source_merge_data                       | 源事件设置数据                                               |
| dispatch_source_get_data                         | 获取源事件数据                                               |
| dispatch_resume                                  | 继续                                                         |
| dispatch_suspend                                 | 挂起                                                         |
| dispatch_cancle                                  | 取消                                                         |



## Reference

[1 iOS探索 多线程之GCD应用](https://juejin.cn/post/6844904122068500487)