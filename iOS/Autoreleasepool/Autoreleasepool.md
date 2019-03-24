# Autoreleasepool

`AutoreleasePool`（自动释放池）是OC中的一种内存自动回收机制，它可以延迟加入AutoreleasePool中的变量release的时机。

在没有手加`Autorelease Pool`的情况下， **Autorelease对象是在当前的runloop迭代结束时释放的，而它能够释放的原因是系统在每个runloop迭代中都加入了自动释放池Push和Pop**



## 1.Autoreleasepool结构

编译器会把`@autoreleasepool{}`改写成:

```objective-c
void *ctx = objc_autoreleasePoolPush();
{}中代码
objc_autoreleasePoolPop(ctx);
					

void *objc_autoreleasePoolPush(void){
  return AutoreleasePoolPage::push();
}

void objc_autoreleasePoolPop(void *ctxt){
  AutoreleasePoolPage:pop(ctxt);
}
```

从上述代码可以知道Push,Pop都是操作 `AutoreleasePoolPage`的



### 1.1  AutoreleasePoolPage 结构

```objective-c
class AutoreleasePoolPage {
    magic_t const magic;	//用于对当前 AutoreleasePoolPage 完整性的校验
    id *next;
    pthread_t const thread;		//thread 保存了当前页所在的线程
    AutoreleasePoolPage * const parent;
    AutoreleasePoolPage *child;
    uint32_t const depth;
    uint32_t hiwat;
};
```

**每一个自动释放池都是由一系列的 AutoreleasePoolPage 组成的，并且每一个 AutoreleasePoolPage 的大小都是 4096 字节**



#### 1.1.1 双向链表

自动释放池中的 `AutoreleasePoolPage` 是以**双向链表**的形式连接起来的：

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-24-140349.jpg)

`parent` *和* `child` 就是用来构造双向链表的指针。



#### 1.1.2 自动释放池中的栈

如果我们的一个 `AutoreleasePoolPage` 被初始化在内存的 `0x100816000 ~ 0x100817000`中，它在内存中的结构如下：

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-24-140609.jpg)

`next` 指向了下一个为空的内存地址，如果 `next` 指向的地址加入一个 `object`，它就会如下图所示**移动到下一个为空的内存地址中**：

![image-20190324220807472](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-24-140809.png)



#### POOL_SENTINEL（哨兵对象）

在每个自动释放池初始化调用 `objc_autoreleasePoolPush` 的时候，都会把一个 `POOL_SENTINEL` push 到自动释放池的栈顶，并且返回这个 `POOL_SENTINEL` 哨兵对象。

```objective-c
int main(int argc, const char * argv[]) {
    {
        void * atautoreleasepoolobj = objc_autoreleasePoolPush();

        // do whatever you want

        objc_autoreleasePoolPop(atautoreleasepoolobj);
    }
    return 0;
}
```

*上面的* `atautoreleasepoolobj` *就是一个* `POOL_SENTINEL`。

而当方法 `objc_autoreleasePoolPop` 调用时，就会向自动释放池中的对象发送 `release` 消息，直到第一个 `POOL_SENTINEL`：

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-24-141046.jpg)



## objc_autoreleasePoolPush

```objective-c
static inline id *autoreleaseFast(id obj)
{
   AutoreleasePoolPage *page = hotPage();
   if (page && !page->full()) {
       return page->add(obj);
   } else if (page) {
       return autoreleaseFullPage(obj, page);
   } else {
       return autoreleaseNoPage(obj);
   }
}
```

底层转化为如上代码

`hotPage` *可以理解为当前正在使用的* `AutoreleasePoolPage`。

上述方法分三种情况选择不同的代码执行：

- 有 `hotPage` 并且当前 `page` 不满
  - 调用 `page->add(obj)` 方法将对象添加至 `AutoreleasePoolPage` 的栈中

- 有 `hotPage` 并且当前 `page` 已满
  - 调用 `autoreleaseFullPage` 初始化一个新的页
  - 调用 `page->add(obj)` 方法将对象添加至 `AutoreleasePoolPage` 的栈

- 无 `hotPage`
  - 调用 `autoreleaseNoPage` 创建一个 `hotPage`
  - 调用 `page->add(obj)` 方法将对象添加至 `AutoreleasePoolPage` 的栈中

### objc_autoreleasePoolPop

作用如上图

栈中存放的指针指向加入需要release的对象或者POOL_SENTINEL（哨兵对象，用于分隔`Autoreleasepool`）。
栈中指向POOL_SENTINEL的指针就是`Autoreleasepool`的一个标记。当`Autoreleasepool`进行出栈操作，每一个比这个哨兵对象后进栈的对象都会release。



![image-20190324222842393](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-24-142843.png)



## 2. Runloop & Autoreleasepool

我个人认为将 `AutoReleasepool` 、 `ARC` 、 `Runloop` 3种技术联系在一起来看是一种全面的认知.

### 2.1Runloop 与 Autoreleasepool 创建

 每个`Runloop`中都会创建一个 `AutoReleasepool` 并在 `Runloop迭代结束`进行释放。何为 `迭代结束`？当前`Runloop` 进入 `Sleep mode`的时候,就结束当前 `Runloop`迭代.新的一轮`Runloop`创建一个新的 `AutoReleasepool`, `Pool`里面的临时对象在结束后得到释放(不一定即时,也有可能延后,系统决定)

 `Runloop`第一个 Observer 监视的事件是 Entry(即将进入Loop)，其回调内会调用 `_objc_autoreleasePoolPush()` 创建自动释放池。其 order 是-2147483647，优先级最高，保证创建释放池发生在其他所有回调之前。

 第二个 Observer 监视了两个事件： BeforeWaiting(准备进入休眠) 时调用`_objc_autoreleasePoolPop()` 和 `_objc_autoreleasePoolPush()` 释放旧的池并创建新池；Exit(即将退出Loop) 时调用 `_objc_autoreleasePoolPop()` 来释放自动释放池。这个 Observer 的 order 是 2147483647，优先级最低，保证其释放池子发生在其他所有回调之后。



### 2.2线程与runloop

线程和 RunLoop 之间是一一对应的，其关系是保存在一个全局的 Dictionary 里。线程刚创建时并没有 RunLoop，如果你不主动获取，那它一直都不会有。RunLoop 的创建是发生在第一次获取时，RunLoop 的销毁是发生在线程结束时。你只能在一个线程的内部获取其 RunLoop（主线程除外）。



## 手动@autoreleasepool 与 嵌套

嵌套`autorelesepool`很好解释,pop的时候总会释放到上次push的位置为止，多层的pool就是多个哨兵对象而已，就像剥洋葱一样，每次一层，互不影响。

手动`autoreleasepool`,如下文参考2例子,可以得知这个`for`循环中，每一次循环会清理掉一次内存,因为完全执行完 `for`循环才会，`runloop`才会进行休眠，如果说是按照系统的`autoreleasepool`来说，应该是休眠前才释放，但是，文中demo内存并没有显示出循环中内存暴涨，这也说明了，**手动autorelesepool 不是在内存峰值时候释放**



## Reference

[1.自动释放池的前世今生 ---- 深入解析 autoreleasepool](https://draveness.me/autoreleasepool)