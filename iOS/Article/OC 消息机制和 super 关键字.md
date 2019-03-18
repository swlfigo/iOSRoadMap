>  原文地址 https://juejin.im/post/5c87a218f265da2dd868cfcd

原文链接 [OC 消息机制和 super 关键字](https://link.juejin.im?target=https%3A%2F%2Fwww.neroxie.com%2F2019%2F03%2F12%2FOC%25E6%25B6%2588%25E6%2581%25AF%25E6%259C%25BA%25E5%2588%25B6%25E5%2592%258Csuper%25E5%2585%25B3%25E9%2594%25AE%25E5%25AD%2597%2F%23more)

## 消息发送

在 Objective-C 里面调用一个方法`[object method]`，运行时会将它翻译成`objc_msgSend(id self, SEL op, ...)`的形式。

### objc_msgSend

`objc_msgSend`的实现在`objc-msg-arm.s`、`objc-msg-arm64.s`等文件中，是通过汇编实现的。这里主要看在`arm64`即`objc-msg-arm64.s`的实现。由于汇编不熟，里面的实现只能连看带猜。

```
	ENTRY _objc_msgSend
	UNWIND _objc_msgSend, NoFrame
	MESSENGER_START

	cmp	x0, #0			// nil check and tagged pointer check
	b.le	LNilOrTagged		//  (MSB tagged pointer looks negative)
	ldr	x13, [x0]		// x13 = isa
	and	x16, x13, #ISA_MASK	// x16 = class	
LGetIsaDone:
	CacheLookup NORMAL		// calls imp or objc_msgSend_uncached

LNilOrTagged:
    /* nil check，如果为空就是调用LReturnZero，LReturnZero里调用MESSENGER_END_NIL*/
	b.eq	LReturnZero		// nil check

	// tagged
	mov	x10, #0xf000000000000000
	cmp	x0, x10
	b.hs	LExtTag
	adrp	x10, _objc_debug_taggedpointer_classes@PAGE
	add	x10, x10, _objc_debug_taggedpointer_classes@PAGEOFF
	ubfx	x11, x0, #60, #4
	ldr	x16, [x10, x11, LSL #3]
	b	LGetIsaDone

LExtTag:
	// ext tagged
	adrp	x10, _objc_debug_taggedpointer_ext_classes@PAGE
	add	x10, x10, _objc_debug_taggedpointer_ext_classes@PAGEOFF
	ubfx	x11, x0, #52, #8
	ldr	x16, [x10, x11, LSL #3]
	b	LGetIsaDone

LReturnZero:
	// x0 is already zero
	mov	x1, #0
	movi	d0, #0
	movi	d1, #0
	movi	d2, #0
	movi	d3, #0
	MESSENGER_END_NIL
	ret

	END_ENTRY _objc_msgSend
复制代码
```

上面的流程可能是这样的：

 ![](https://sylarimage.oss-cn-shenzhen.aliyuncs.com/20190318140410.png)



从`CacheLookup`的注释有两处：

1.  `calls imp or objc_msgSend_uncached`
2.  `Locate the implementation for a selector in a class method cache.`

即使看不懂汇编代码，但是从上面的注释我们可以猜测，消息机制会先从缓存中去查找。

### __objc_msgSend_uncached

通过方法名我们可以知道，没有缓存的时候应该会执行`__objc_msgSend_uncached`。

```
	STATIC_ENTRY __objc_msgSend_uncached
	UNWIND __objc_msgSend_uncached, FrameWithNoSaves

	// THIS IS NOT A CALLABLE C FUNCTION
	// Out-of-band x16 is the class to search

	MethodTableLookup
	br	x17

	END_ENTRY __objc_msgSend_uncached
复制代码
```

这里的`MethodTableLookup`里涉及到`objc-runtime-new.mm`文件中的`_class_lookupMethodAndLoadCache3`。该函数会调用`lookUpImpOrForward`函数。

### lookUpImpOrForward

`lookUpImpOrForward`会返回一个`imp`，它的函数实现比较长，但是注释写的非常清楚。它的实现主要由以下几步（这里直接从缓存获取开始）：

1.  通过`cache_getImp`从缓存中获取方法，有则返回，否则进入第 2 步；
2.  通过`getMethodNoSuper_nolock`从类的方法列表中获取，有加入缓存中并返回，否则进入第 3 步；
3.  通过父类的缓存和父类的方法列表中寻找是否有对应的 imp，此时会进入一个`for`循环，沿着类的父类一直往上找，直接找到 NSObject 为止。如果找到返回，否则进入第 4 步；
4.  进入方法决议（method resolve）的过程即调用`_class_resolveMethod`，如果失败，进入第 5 步；
5.  在缓存、当前类、父类以及方法决议都没有找到的情况下，Objective-C 还为我们提供了最后一次翻身的机会，调用`_objc_msgForward_impcache`进行方法转发，如果找到便加入缓存；如果没有就 crash。

上述过程中有几个比较重要的函数：

#### **_class_resolveMethod**

```
void _class_resolveMethod(Class cls, SEL sel, id inst) {
    if (! cls->isMetaClass()) {
        // try [cls resolveInstanceMethod:sel]
        _class_resolveInstanceMethod(cls, sel, inst);
    } 
    else {
        // try [nonMetaClass resolveClassMethod:sel]
        // and [cls resolveInstanceMethod:sel]
        _class_resolveClassMethod(cls, sel, inst);
        if (!lookUpImpOrNil(cls, sel, inst, 
                            NO/*initialize*/, YES/*cache*/, NO/*resolver*/)) 
        {
            _class_resolveInstanceMethod(cls, sel, inst);
        }
    }
}
复制代码
```

上述函数会根据当前传入的类的是不是一个元类，在`_class_resolveInstanceMethod`和`_class_resolveClassMethod`中选择一个进行调用。注释也说明了这两个方法的作用就是判断当前类是否实现了 `resolveInstanceMethod:`或者`resolveClassMethod:`方法，然后用`objc_msgSend`执行上述方法。

#### **_class_resolveClassMethod**

`_class_resolveClassMethod`和`_class_resolveInstanceMethod`实现类似，这里就只看`_class_resolveClassMethod`的实现。

```
static void _class_resolveClassMethod(Class cls, SEL sel, id inst) {
    assert(cls->isMetaClass());

    if (! lookUpImpOrNil(cls, SEL_resolveClassMethod, inst, 
                         NO/*initialize*/, YES/*cache*/, NO/*resolver*/)) {
         //没有找到resolveClassMethod方法，直接返回。
        return;
    }

    BOOL (*msg)(Class, SEL, SEL) = (typeof(msg))objc_msgSend;
    bool resolved = msg(_class_getNonMetaClass(cls, inst), 
                        SEL_resolveClassMethod, sel);

    // 缓存结果
    IMP imp = lookUpImpOrNil(cls, sel, inst, 
                             NO/*initialize*/, YES/*cache*/, NO/*resolver*/);
    // 以下代码省略不影响阅读                          
}
复制代码
```

#### **_objc_msgForward_impcache**

```
	STATIC_ENTRY __objc_msgForward_impcache

	MESSENGER_START
	nop
	MESSENGER_END_SLOW

	// No stret specialization.
	b	__objc_msgForward

	END_ENTRY __objc_msgForward_impcache

	ENTRY __objc_msgForward

	adrp	x17, __objc_forward_handler@PAGE
	ldr	x17, [x17, __objc_forward_handler@PAGEOFF]
	br	x17

	END_ENTRY __objc_msgForward
复制代码
```

`_objc_msgForward_impcache`用来进行消息转发，但是其真正的核心是调用`_objc_msgForward`。

### 消息转发

关于`_objc_msgForward`在`objc`中并没有其相关实现，只能看到`_objc_forward_handler`。其实`_objc_msgForward`的实现是在`CFRuntime.c`中的，但是开源出来的`CFRuntime.c`并没有相关实现，但是也不影响我们对真理的追求。

我们做几个实验来验证消息转发。

#### 消息重定向测试

```
// .h文件
@interface AObject : NSObject

- (void)sendMessage;

@end
// .m文件
@implementation AObject

/** 验证消息重定向 */
- (id)forwardingTargetForSelector:(SEL)aSelector {
    if (aSelector == @selector(sendMessage)) {
         return [BObject new];
    }

    return [super forwardingTargetForSelector:aSelector];
}

@end

// .h文件
@interface BObject : NSObject

- (void)sendMessage;

@end

// .m文件
@implementation BObject

- (void)sendMessage {
    NSLog(@"%@ send message", self.class);
}

@end

// 调用
AObject *a = [AObject new];
[a sendMessage];
复制代码
```

运行结果：

```
2019-03-12 10:18:54.252949+0800 iOSCodeLearning[18165:5967575] BObject send message
复制代码
```

在`forwardingTargetForSelector:`处打个断点，查看一下调用栈：

![](https://sylarimage.oss-cn-shenzhen.aliyuncs.com/20190318140550.png)

`_CF_forwarding_prep_0`和`___forwarding___`这两个方法会先被调用了，之后调用了`forwardingTargetForSelector:`。

#### 方法签名测试

```
// .h文件
@interface AObject : NSObject

- (void)sendMessage;

@end
// .m文件
@implementation AObject

/** 消息重定向 */
- (id)forwardingTargetForSelector:(SEL)aSelector {
   return nil;
}

/** 方法签名测试 */
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {
    if (aSelector == @selector(sendMessage)) {
        return [BObject instanceMethodSignatureForSelector:@selector(sendMessage)];
    }

    return [super methodSignatureForSelector:aSelector];
}

- (void)forwardInvocation:(NSInvocation *)anInvocation {
    SEL selector = [anInvocation selector];
    if (selector == @selector(sendMessage)) {
        [anInvocation invokeWithTarget:[BObject new]];
    } else {
        [super forwardInvocation:anInvocation];
    }
}

@end

// .h文件
@interface BObject : NSObject

- (void)sendMessage;

@end

// .m文件
@implementation BObject

- (void)sendMessage {
    NSLog(@"%@ send message", self.class);
}

@end

// 调用
AObject *a = [AObject new];
[a sendMessage];
复制代码
```

![](https://sylarimage.oss-cn-shenzhen.aliyuncs.com/20190318140550.png)

代码执行结果和消息重定向测试的运行结果一致。`_CF_forwarding_prep_0`和`___forwarding___`这两个方法又再次被调用了，之后代码会先执行`forwardingTargetForSelector:`（消息重定向），消息重定向如果失败后调用`methodSignatureForSelector:`和`forwardInvocation:`方法签名。所以说`___forwarding___`方法才是消息转发的真正实现。

#### crash 测试

```
// .h文件
@interface AObject : NSObject

- (void)sendMessage;

@end
// .m文件
@implementation AObject

- (id)forwardingTargetForSelector:(SEL)aSelector {
    return nil;
}

- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {
    return [super methodSignatureForSelector:aSelector];
}

- (void)forwardInvocation:(NSInvocation *)anInvocation {
}

/** 验证Crash */
- (void)doesNotRecognizeSelector:(SEL)aSelector {
    if (aSelector == @selector(sendMessage)) {
        NSLog(@"%@ doesNotRecognizeSelector", self.class);
    }
}

@end

// .h文件
@interface BObject : NSObject

- (void)sendMessage;

@end

// .m文件
@implementation BObject

- (void)sendMessage {
    NSLog(@"%@ send message", self.class);
}

@end

// 调用
AObject *a = [AObject new];
[a sendMessage];
复制代码
```

代码运行结果肯定是 crash，结合上面的代码我们知道消息转发会调用`___forwarding___`这个内部方法。`___forwarding___`方法调用顺序是`forwardingTargetForSelector:`->`methodSignatureForSelector:`->`doesNotRecognizeSelector:`

我们用一张图表示整个消息发送的过程：

![](https://sylarimage.oss-cn-shenzhen.aliyuncs.com/20190318140632.png)

## super 关键字

我们先查看一下执行`[super init]`的时候，调用了那些方法

![](https://sylarimage.oss-cn-shenzhen.aliyuncs.com/20190318140712.png)

`objc_msgSendSuper2`的声明在`objc-abi.h`中

```
// objc_msgSendSuper2() takes the current search class, not its superclass.
OBJC_EXPORT id _Nullable
objc_msgSendSuper2(struct objc_super * _Nonnull super, SEL _Nonnull op, ...)
    OBJC_AVAILABLE(10.6, 2.0, 9.0, 1.0, 2.0);
复制代码
```

`objc_super`的定义如下：

```
struct objc_super {
    /// Specifies an instance of a class.
    __unsafe_unretained _Nonnull id receiver;

    /// Specifies the particular superclass of the instance to message. 
#if !defined(__cplusplus)  &&  !__OBJC2__
    /* For compatibility with old objc-runtime.h header */
    __unsafe_unretained _Nonnull Class class;
#else
    __unsafe_unretained _Nonnull Class super_class;
#endif
    /* super_class is the first class to search */
};
复制代码
```

从上面的定义我们可以知道`receiver`即消息的实际接收者， `super_class`为指向当前类的父类。

所以该函数实际的操作是：从`objc_super`结构体指向的`super_class`开始查找，直到会找到 NSObject 的方法为止。找到后以`receiver`去调用。当然整个查找的过程还是和消息发送的流程一样。

所以我们能理解为什么下面这段代码执行的结果都是`AObject`了吧。虽然使用`[super class]`，但是真正执行方法的对象还是`AObject`。

```
// 代码
@implementation AObject

- (instancetype)init {
    if (self = [super init]) {
        NSLog(@"%@", [super class]);
        NSLog(@"%@", [self class]);
    }

    return self;
}

@end

// 执行结果
2019-03-12 19:44:46.003313+0800 iOSCodeLearning[34431:7234182] AObject
2019-03-12 19:44:46.003442+0800 iOSCodeLearning[34431:7234182] AObject
复制代码
```