# GCDThread

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



![img](https://user-gold-cdn.xitu.io/2020/4/7/17154910bcb7fec0?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



> 主队列和全局队列单独考虑，组合结果以总结表格为准

#### 1.串行+同步

任务一个接一个执行，不开辟线程

```
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
复制代码
```

#### 2.串行+异步

任务一个接一个执行，会开辟线程

```
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
复制代码
```

#### 3.并发+同步

任务一个接一个执行，不开辟线程

```
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
复制代码
```

#### 4.并发+异步

任务乱序执行，开辟线程

```
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
复制代码
```

下面来看一下`主队列`和`全局队列`的使用情况：

#### 5.主队列+同步

相互等待，造成死锁

```
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
复制代码
```

#### 6.主队列+异步

任务一个接一个执行，不开辟线程

```
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
复制代码
```

#### 7.全局队列+同步

任务一个接一个执行，不开辟线程（同并发+同步）

```
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
复制代码
```

#### 8.全局队列+异步

任务乱序执行，开辟线程（同并发+异步）

```
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
复制代码
```

**总结一下：**

| 执行\队列 | 串行队列             | 并发队列             | 主队列               | 全局队列             |
| --------- | -------------------- | -------------------- | -------------------- | -------------------- |
| 同步执行  | 按序执行，不开辟线程 | 按序执行，不开辟线程 | 死锁                 | 按序执行，不开辟线程 |
| 异步执行  | 按序执行，开辟线程   | 乱序执行，开辟线程   | 按序执行，不开辟线程 | 乱序执行，开辟线程   |

## 


作者：我是好宝宝
链接：https://juejin.cn/post/6844904122068500487
来源：掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。