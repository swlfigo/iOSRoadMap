# Autoreleasepool

`AutoreleasePool`（自动释放池）是OC中的一种内存自动回收机制，它可以延迟加入AutoreleasePool中的变量release的时机。

在没有手加`Autorelease Pool`的情况下， **Autorelease对象是在当前的runloop迭代结束时释放的，而它能够释放的原因是系统在每个runloop迭代中都加入了自动释放池Push和Pop**



## Autoreleasepool结构

编译器会把`@autoreleasepool{}`改写成:

```objective-c
struct __AtAutoreleasePool {
    //构造函数-->可以类比成OC的init方法，在创建时调用
  __AtAutoreleasePool()
    {
        atautoreleasepoolobj = objc_autoreleasePoolPush();
    }
    
    //析构函数-->可以类比成OC的dealloc方法，在销毁时调用
  ~__AtAutoreleasePool()
    {
        objc_autoreleasePoolPop(atautoreleasepoolobj);
    }
    
  void * atautoreleasepoolobj;
};



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



单层`@autoreleasepool {}`的情况，那么如果有多层`@autoreleasepool {}`嵌套在一起，就可以按照同样的规则来拆解

![img](https://sylarimage.oss-cn-shenzhen.aliyuncs.com/uPic/78d3bc4ded3341bdbe74daa6265a63b7~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)





##  AutoreleasePoolPage 结构



```objective-c
class AutoreleasePoolPage {
    magic_t const magic;	//用于对当前 AutoreleasePoolPage 完整性的校验
    id *next;	//指向AutoreleasePoolPage内下一个可以用来存放自动释放对象的内存地址
    pthread_t const thread;		//thread 保存了当前页所在的线程,自动释放池所属的线程，说明它不能跟多个线程关联。
    AutoreleasePoolPage * const parent; //指向上一页释放池的指针
    AutoreleasePoolPage *child;	//指向下一页释放池的指针
    uint32_t const depth;
    uint32_t hiwat;
};
```



![AutoreleasePoolPage的begin()和end()](https://sylarimage.oss-cn-shenzhen.aliyuncs.com/uPic/8524da26486945268aa5330ffd635d3b~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)





**每一个自动释放池都是由一系列的 AutoreleasePoolPage 组成的，并且每一个 AutoreleasePoolPage 的大小都是 4096 字节**

- AutoreleasePool并没有特定的内存结构，它是通过以`AutoreleasePoolPage`为节点的双向链表。
- 每一个`AutoreleasePoolPage`节点是一个堆栈结，且大小为4096个字节。
- 一个`AutoreleasePoolPage`节点对应着一个线程，属于一一对应关系。

AutoreleasePool结构如图所示：



![AutoreleasePoolPage结构示意图](https://sylarimage.oss-cn-shenzhen.aliyuncs.com/uPic/cecf2e88ccbd4110b8175fee16d7dab2~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)



### 双向链表

自动释放池中的 `AutoreleasePoolPage` 是以**双向链表**的形式连接起来的：

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-24-140349.jpg)

`parent` *和* `child` 就是用来构造双向链表的指针。



接着我们看一下`AutoreleasePoolPage`的构造函数以及一些操作方法：

```c++
    //构造函数
    AutoreleasePoolPage(AutoreleasePoolPage *newParent) 
        : magic(), next(begin()), thread(pthread_self()),
          parent(newParent), child(nil), 
          depth(parent ? 1+parent->depth : 0), 
          hiwat(parent ? parent->hiwat : 0)
    { 
        if (parent) {
            parent->check();
            assert(!parent->child);
            parent->unprotect();
            parent->child = this;
            parent->protect();
        }
        protect();
    }
    
    //相关操作方法
    id * begin() {
        return (id *) ((uint8_t *)this+sizeof(*this));
    }

    id * end() {
        return (id *) ((uint8_t *)this+SIZE);
    }

    bool empty() {
        return next == begin();
    }

    bool full() { 
        return next == end();
    }

    bool lessThanHalfFull() {
        return (next - begin() < (end() - begin()) / 2);
    }

    id *add(id obj)
    {
        assert(!full());
        unprotect();
        id *ret = next;  // faster than `return next-1` because of aliasing
        *next++ = obj;
        protect();
        return ret;
    }

```

- begin() 表示了一个`AutoreleasePoolPage`节点开始存autorelease对象的位置。
- end() 一个`AutoreleasePoolPage`节点最大的位置
- empty() 如果`next`指向beigin()说明为空
- full() 如果`next`指向end)说明满了
- id *add(id obj) 添加一个autorelease对象，next指向下一个存对象的地址。

所以一个空的`AutoreleasePoolPage`的结构如下：

![AutoreleasePoolPage](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-03-05-075438.jpg)

### AutoreleasePoolPage::push()

push代码如下：

```c++
    static inline void *push() 
    {
        id *dest;
        if (DebugPoolAllocation) {
            // Each autorelease pool starts on a new pool page.
            dest = autoreleaseNewPage(POOL_BOUNDARY);
        } else {
            dest = autoreleaseFast(POOL_BOUNDARY);
        }
        assert(dest == EMPTY_POOL_PLACEHOLDER || *dest == POOL_BOUNDARY);
        return dest;
    }

```

push执行的时候首先会进行判断，如果是需要每个pool都生成一个新page，即`DebugPoolAllocation`为`YES`，则执行`autoreleaseNewPage`方法，否则执行`autoreleaseFast`方法。

#### autoreleaseNewPage

`autoreleaseNewPage`分为两种情况：

1. 当前存在page执行`autoreleaseFullPage`方法；
2. 当前不存在page`autoreleaseNoPage`方法。

#### autoreleaseFast

`autoreleaseFast`分为三种情况：

1. 存在page且未满，通过`add()`方法进行添加；
2. 当前page已满执行`autoreleaseFullPage`方法；
3. 当前不存在page执行`autoreleaseNoPage`方法。

### hotPage

前面讲到的page其实就是`hotPage`，通过`AutoreleasePoolPage *page = hotPage();`获取。

```c++
    static inline AutoreleasePoolPage *hotPage() 
    {
        AutoreleasePoolPage *result = (AutoreleasePoolPage *)
            tls_get_direct(key);
        if ((id *)result == EMPTY_POOL_PLACEHOLDER) return nil;
        if (result) result->fastcheck();
        return result;
    }
```



通过上面的代码我们知道当前页是存在`TLS（线程私有数据）`里面的。所以说**第一次调用push的时候，没有page自然连hotPage也没有**。

#### autoreleaseFullPage

```c++
static __attribute__((noinline))
    id *autoreleaseFullPage(id obj, AutoreleasePoolPage *page)
    {
        // The hot page is full. 
        // Step to the next non-full page, adding a new page if necessary.
        // Then add the object to that page.
        assert(page == hotPage());
        assert(page->full()  ||  DebugPoolAllocation);

        do {
            if (page->child) page = page->child;
            else page = new AutoreleasePoolPage(page);
        } while (page->full());

        setHotPage(page);
        return page->add(obj);
    }
```

`autoreleaseFullPage`会从传入的`page`开始遍历整个双向链表，如果`page`满了，就看它的`child`节点，直到查找到一个未满的`AutoreleasePoolPage`。接着使用`AutoreleasePoolPage`构造函数传入`parent`创建一个新的`AutoreleasePoolPage`的节点（此时跳出了while循环）。

在查找到一个可以使用的`AutoreleasePoolPage`之后，会将该页面标记成`hotPage`，然后调动`add()`方法添加对象。

#### autoreleaseNoPage

```c++
static __attribute__((noinline))
    id *autoreleaseNoPage(id obj)
    {
        //"no page"意味着没有没有池子被push或者说push了一个空的池子
        assert(!hotPage());

        bool pushExtraBoundary = false;
        if (haveEmptyPoolPlaceholder()) {//push了一个空的池子
            pushExtraBoundary = true;
        }
        else if (obj != POOL_BOUNDARY  &&  DebugMissingPools) {
            _objc_inform("MISSING POOLS: (%p) Object %p of class %s "
                         "autoreleased with no pool in place - "
                         "just leaking - break on "
                         "objc_autoreleaseNoPool() to debug", 
                         pthread_self(), (void*)obj, object_getClassName(obj));
            objc_autoreleaseNoPool(obj);
            return nil;
        }
        else if (obj == POOL_BOUNDARY  &&  !DebugPoolAllocation) {
            //没有池子被push
            return setEmptyPoolPlaceholder();
        }

        AutoreleasePoolPage *page = new AutoreleasePoolPage(nil);
        setHotPage(page);
        
        if (pushExtraBoundary) {
            //push了一个空的池子，添加哨兵对象
            page->add(POOL_BOUNDARY);
        }
    
        return page->add(obj);
    }
    
    //haveEmptyPoolPlaceholder的本质
        static inline bool haveEmptyPoolPlaceholder()
    {
        id *tls = (id *)tls_get_direct(key);
        return (tls == EMPTY_POOL_PLACEHOLDER);
    }

```

从上面的代码我们可以知道，既然当前内存中不存在`AutoreleasePoolPage`，就要从头开始构建这个自动释放池的双向链表，也就是说，新的`AutoreleasePoolPage`是没有`parent`指针的。

初始化之后，将当前页标记为`hotPage`，然后会先向这个`page`中添加一个`POOL_BOUNDARY`的标记，来确保在`pop`调用的时候，不会出现异常。

最后，将`obj`添加到自动释放池中。

#### autorelease方法

接着看一下当对象调用`autorelase`方法发生了什么。

```c++
- (id)autorelease {
    return ((id)self)->rootAutorelease();
}

inline id 
objc_object::rootAutorelease()
{
    if (isTaggedPointer()) return (id)this;
    if (prepareOptimizedReturn(ReturnAtPlus1)) return (id)this;

    return rootAutorelease2();
}

__attribute__((noinline,used))
id 
objc_object::rootAutorelease2()
{
    assert(!isTaggedPointer());
    return AutoreleasePoolPage::autorelease((id)this);
}

static inline id autorelease(id obj)
{
    assert(obj);
    assert(!obj->isTaggedPointer());
    id *dest __unused = autoreleaseFast(obj);
    assert(!dest  ||  dest == EMPTY_POOL_PLACEHOLDER  ||  *dest == obj);
    return obj;
}
```

从上面的源码我们看到，对象调用`autorelase`方法，最后会变成`AutoreleasePoolPage`的`autorelease`函数。`AutoreleasePoolPage`的`autorelease`的本质就是调用`autoreleaseFast(obj)`函数。只不过`push`操作插入的是一个`POOL_BOUNDARY` ，而`autorelease`操作插入的是一个具体的`autoreleased`对象即`AutoreleasePoolPage`入栈操作。

当然这么说并不严谨，因为我们需要考虑是否是`Tagged Pointer`和是否进行优化的情况（`prepareOptimizedReturn`这个后面也会提到），如果不满足这两个条件才会进入缓存池。

所以push的流程是：

![AutoreleasePoolPush流程](https://user-gold-cdn.xitu.io/2019/2/27/1692d6298f6722a9?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



### AutoreleasePoolPage::pop(ctxt)

```c++
    static inline void pop(void *token) 
    {
        AutoreleasePoolPage *page;
        id *stop;

        //第一种情况：autoreleasepool首次push的时候返回的，也就是最顶层的page执行pop会执行这一部分
        if (token == (void*)EMPTY_POOL_PLACEHOLDER) {
            // Popping the top-level placeholder pool.
            if (hotPage()) {
                // Pool was used. Pop its contents normally.
                // Pool pages remain allocated for re-use as usual.
                pop(coldPage()->begin());
            } else {
                // Pool was never used. Clear the placeholder.
                setHotPage(nil);
            }
            return;
        }

        page = pageForPointer(token);
        
        //https://stackoverflow.com/questions/24952549/does-nsthread-create-autoreleasepool-automatically-now
        //第二种情况：在非ARC的情况下，在新创建的线程中不使用autoreleasepool，直接调用autorelease方法时会出现这个情况。此时没有pool，直接进行autorelease。
        stop = (id *)token;
        if (*stop != POOL_BOUNDARY) {
            if (stop == page->begin()  &&  !page->parent) {
                // Start of coldest page may correctly not be POOL_BOUNDARY:
                // 1. top-level pool is popped, leaving the cold page in place
                // 2. an object is autoreleased with no pool
            } else {
                // Error. For bincompat purposes this is not 
                // fatal in executables built with old SDKs.
                return badPop(token);
            }
        }

        if (PrintPoolHiwat) printHiwat();
        //第三种情况：也就是我们经常碰到的情况
        page->releaseUntil(stop);

        // memory: delete empty children
        if (DebugPoolAllocation  &&  page->empty()) {
            // special case: delete everything during page-per-pool debugging
            AutoreleasePoolPage *parent = page->parent;
            page->kill();
            setHotPage(parent);
        } else if (DebugMissingPools  &&  page->empty()  &&  !page->parent) {
            // special case: delete everything for pop(top) 
            // when debugging missing autorelease pools
            page->kill();
            setHotPage(nil);
        } 
        else if (page->child) {
            // hysteresis: keep one empty child if page is more than half full
            if (page->lessThanHalfFull()) {
                page->child->kill();
            }
            else if (page->child->child) {
                page->child->child->kill();
            }
        }
    }
```

这里我们主要分析下第三种情况。

### releaseUntil

```c++
void releaseUntil(id *stop) {
    while (this->next != stop) {
        AutoreleasePoolPage *page = hotPage();

        while (page->empty()) {
            page = page->parent;
            setHotPage(page);
        }

        page->unprotect();
        id obj = *--page->next;
        memset((void*)page->next, SCRIBBLE, sizeof(*page->next));
        page->protect();

        if (obj != POOL_BOUNDARY) {
            objc_release(obj);
        }
    }

    setHotPage(this);
}
```

从next指针开始，一个一个向前调用`objc_release`，直到碰到push时压入的pool为止。

所以autoreleasePool的运行过程应该是：

```c++
pool1 = push()
...
    pool2 = push()
    ...
        pool3 = push()
        ...
        pop(pool3)
    ...
    pop(pool2)
...
pop(pool1)

```

每次pop，实际上都会把最近一次push之后添加进去的对象全部release掉。





#### 自动释放池中的栈

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









## Runloop 与 Autoreleasepool 创建

每一个线程都会维护自己的autoreleasePool堆栈，也就是说每一个autoreleasePool对应一个线程。 

`@autoreleasepool{}`的作用，实际上就是在作用域的头和尾分别调用了`objc_autoreleasePoolPush();`和`objc_autoreleasePoolPop()`函数



~~每个`Runloop`中都会创建一个 `AutoReleasepool` 并在 `Runloop迭代结束`进行释放。~~何为 `迭代结束`？当前`Runloop` 进入 `Sleep mode`的时候,就结束当前 `Runloop`迭代.新的一轮`Runloop`创建一个新的 `AutoReleasepool`, `Pool`里面的临时对象在结束后得到释放(不一定即时,也有可能延后,系统决定)

 `Runloop`第一个 Observer 监视的事件是 Entry(即将进入Loop)，其回调内会调用 `_objc_autoreleasePoolPush()` 创建自动释放池。其 order 是-2147483647，优先级最高，保证创建释放池发生在其他所有回调之前。

 第二个 Observer 监视了两个事件： BeforeWaiting(准备进入休眠) 时调用`_objc_autoreleasePoolPop()` 和 `_objc_autoreleasePoolPush()` 释放旧的池并创建新池；Exit(即将退出Loop) 时调用 `_objc_autoreleasePoolPop()` 来释放自动释放池。这个 Observer 的 order 是 2147483647，优先级最低，保证其释放池子发生在其他所有回调之后。



## 总结

1.**子线程在使用autorelease对象时，如果没有autoreleasepool会在autoreleaseNoPage中懒加载一个出来。**

2.在runloop的run:beforeDate，以及一些source的callback中，有autoreleasepool的push和pop操作，总结就是系统在很多地方都差不多autorelease的管理操作。

3.就算插入没有pop也没关系，在线程exit的时候会释放资源，执行AutoreleasePoolPage::tls_dealloc，在这里面会清空autoreleasepool。



## 手动@autoreleasepool 与 嵌套

嵌套`autorelesepool`很好解释,pop的时候总会释放到上次push的位置为止，多层的pool就是多个哨兵对象而已，就像剥洋葱一样，每次一层，互不影响。

手动`autoreleasepool`,如下文参考2例子,可以得知这个`for`循环中，每一次循环会清理掉一次内存,因为完全执行完 `for`循环才会，`runloop`才会进行休眠，如果说是按照系统的`autoreleasepool`来说，应该是休眠前才释放，但是，文中demo内存并没有显示出循环中内存暴涨，这也说明了，**手动autorelesepool 不是在内存峰值时候释放**



## Reference

[1.自动释放池的前世今生 ---- 深入解析 autoreleasepool](https://draveness.me/autoreleasepool)

[2. 解密Runloop](http://mrpeak.cn/blog/ios-runloop/)

[3. 在ARC环境中autoreleasepool(runloop)的研究](https://juejin.im/post/59eabe2451882578ca2dc145)

[4. 黑幕背后的Autorelease](https://blog.sunnyxx.com/2014/10/15/behind-autorelease/)

[5. iOS RunLoop详解](https://juejin.im/post/5aca2b0a6fb9a028d700e1f8)

[6.深入了解Runloop](https://blog.ibireme.com/2015/05/18/runloop/)

[7.带着问题看源码----子线程AutoRelease对象何时释放](https://suhou.github.io/2018/01/21/%E5%B8%A6%E7%9D%80%E9%97%AE%E9%A2%98%E7%9C%8B%E6%BA%90%E7%A0%81----%E5%AD%90%E7%BA%BF%E7%A8%8BAutoRelease%E5%AF%B9%E8%B1%A1%E4%BD%95%E6%97%B6%E9%87%8A%E6%94%BE/)

[8.AutoreleasePool的实现](https://juejin.cn/post/6844903783818854414)

[内存管理剖析（四）——autorelease原理分析经历过MRC时代的开发者，肯定都用过autorelease方法，用于 - 掘金 (juejin.cn)](https://juejin.cn/post/6966596279228973086?searchId=2024090809575838D9FFC4087BA1803C84)