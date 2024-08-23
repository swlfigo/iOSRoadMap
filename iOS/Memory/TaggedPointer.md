

# Tagged Pointer



Tagged Pointer是苹果在64bit设备提出的一种存储小对象的技术，用于优化NSNumber、NSDate、NSString等小对象的储存

主要解决 **内存浪费** 和 **访问效率** 的问题

它具有以下特点

- Tagged Pointer指针的值不再是地址了，而是真正的值。所以，实际上它不再是一个对象了，它只是一个披着对象皮的普通变量而已。
- 它的内存并不存储在堆中，也不需要 malloc 和 free，不走引用计数那一套逻辑，由系统来处理释放
- 可以通过设置环境变量`OBJC_DISABLE_TAGGED_POINTERS`来有开发者决定是否使用这项技术
- 专门用于储存小对象



**未引入Tagged Pointer**

![img](https://sylarimage.oss-cn-shenzhen.aliyuncs.com/uPic/4ffec3cac54d4d5988ffea716c17c021~tplv-k3u1fbpfcp-zoom-1.png)

**引入Tagged Pointer**

![img](https://sylarimage.oss-cn-shenzhen.aliyuncs.com/uPic/9ce0544f0bb14364bf1adc63b0cde0ed~tplv-k3u1fbpfcp-zoom-1.png)




从32位迁移到64位CPU，逻辑上虽然不会有任何变化，但是所占有的内存空间却会**翻倍**。下面以NSNumber对象为例，大家可以清晰看出NSNumber对象在内存空间上的变化情况：

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2020-10-28-144308.jpg)



## 源码

```powershell
#if (TARGET_OS_OSX || TARGET_OS_IOSMAC) && __x86_64__
    // 64-bit Mac - tag bit is LSB
#   define OBJC_MSB_TAGGED_POINTERS 0
#else
    // Everything else - tag bit is MSB
#   define OBJC_MSB_TAGGED_POINTERS 1 // 最高有效位
#endif

#define _OBJC_TAG_INDEX_MASK 0x7 // 0b111表示有扩展的标记位，扩展标记位占8位
// array slot includes the tag bit itself
#define _OBJC_TAG_SLOT_COUNT 16
#define _OBJC_TAG_SLOT_MASK 0xf // 0b1111 taggedpointer + 有扩展标记位的mask

#define _OBJC_TAG_EXT_INDEX_MASK 0xff
// array slot has no extra bits
#define _OBJC_TAG_EXT_SLOT_COUNT 256 // 扩展标记位能表示的个数
#define _OBJC_TAG_EXT_SLOT_MASK 0xff // 0b1111 1111

#if OBJC_MSB_TAGGED_POINTERS
#   define _OBJC_TAG_MASK (1UL<<63) // 是否是tagged pointer的标记位 1表示是 0表示不是
#   define _OBJC_TAG_INDEX_SHIFT 60 // 基础tag位的偏移，从2-4bit，结合_OBJC_TAG_INDEX_MASK来获取到基础tag的值
#   define _OBJC_TAG_SLOT_SHIFT 60
#   define _OBJC_TAG_PAYLOAD_LSHIFT 4 // LSHIFT和RSHIFT配合使用用来对数据移位混淆及恢复
#   define _OBJC_TAG_PAYLOAD_RSHIFT 4
#   define _OBJC_TAG_EXT_MASK (0xfUL<<60) // 1111 0000 ... 0000 0000 是否有扩展标记位 tag位111表示有扩展标记位
#   define _OBJC_TAG_EXT_INDEX_SHIFT 52 // 扩展tag位的偏移，从5-12bit共8位，结合_OBJC_TAG_EXT_INDEX_MASK来获取扩展tag的值
#   define _OBJC_TAG_EXT_SLOT_SHIFT 52
#   define _OBJC_TAG_EXT_PAYLOAD_LSHIFT 12
#   define _OBJC_TAG_EXT_PAYLOAD_RSHIFT 12
#else
#   define _OBJC_TAG_MASK 1UL
#   define _OBJC_TAG_INDEX_SHIFT 1
#   define _OBJC_TAG_SLOT_SHIFT 0
#   define _OBJC_TAG_PAYLOAD_LSHIFT 0
#   define _OBJC_TAG_PAYLOAD_RSHIFT 4
#   define _OBJC_TAG_EXT_MASK 0xfUL
#   define _OBJC_TAG_EXT_INDEX_SHIFT 4
#   define _OBJC_TAG_EXT_SLOT_SHIFT 4
#   define _OBJC_TAG_EXT_PAYLOAD_LSHIFT 0
#   define _OBJC_TAG_EXT_PAYLOAD_RSHIFT 12
#endif

```

定义了很多位信息，我们需要关注的几个：

- _OBJC_TAG_MASK ：标记位标记该指针是否是tagged pointer
- _OBJC_TAG_INDEX_MASK ：tag的值是7表示有扩展的tag位
- 其他的都是一些定义，用来通过位运算来获取tag的值、ext tag的值的mask以及一些其他的左移右移位



##### 如何判断是tagged pointer

有一个标记位来标识指针是否是tagged pointer的

```objectivec
static inline bool 
_objc_isTaggedPointer(const void * _Nullable ptr)
{
    return ((uintptr_t)ptr & _OBJC_TAG_MASK) == _OBJC_TAG_MASK;
}
```

通过位运算获取标识位的值来确定是否是tagged pointer；需要留意的是不同的架构标记位不太一样，有的是用最低位、有的使用最高位。



##### 支持 Tagged Pointer 对象类型

系统通过3bit的标记位来标识tagged pointer对象的类，它的定义在`objc_tag_index_t`中 比如2表示是NSString、6表示是NSDate，我们知道3bit能表示的最大值是7，这个7系统用来预留，用来标记是否有额外的标记位，这样就能支持更多的类支持tagged pointer

```cpp
#if __has_feature(objc_fixed_enum)  ||  __cplusplus >= 201103L
enum objc_tag_index_t : uint16_t
#else
typedef uint16_t objc_tag_index_t;
enum
#endif
{
    // 60-bit payloads
    OBJC_TAG_NSAtom            = 0, 
    OBJC_TAG_1                 = 1, 
    OBJC_TAG_NSString          = 2, 
    OBJC_TAG_NSNumber          = 3, 
    OBJC_TAG_NSIndexPath       = 4, 
    OBJC_TAG_NSManagedObjectID = 5, 
    OBJC_TAG_NSDate            = 6,

    // 60-bit reserved
    OBJC_TAG_RESERVED_7        = 7, 

    // 52-bit payloads
    OBJC_TAG_Photos_1          = 8,
    OBJC_TAG_Photos_2          = 9,
    OBJC_TAG_Photos_3          = 10,
    OBJC_TAG_Photos_4          = 11,
    OBJC_TAG_XPC_1             = 12,
    OBJC_TAG_XPC_2             = 13,
    OBJC_TAG_XPC_3             = 14,
    OBJC_TAG_XPC_4             = 15,
    OBJC_TAG_NSColor           = 16,
    OBJC_TAG_UIColor           = 17,
    OBJC_TAG_CGColor           = 18,
    OBJC_TAG_NSIndexSet        = 19,

    OBJC_TAG_First60BitPayload = 0, 
    OBJC_TAG_Last60BitPayload  = 6, 
    OBJC_TAG_First52BitPayload = 8, 
    OBJC_TAG_Last52BitPayload  = 263, 

    OBJC_TAG_RESERVED_264      = 264
};
#if __has_feature(objc_fixed_enum)  &&  !defined(__cplusplus)
typedef enum objc_tag_index_t objc_tag_index_t;
#endif
```

即针对`NSString`、`NSNumber`、`NSDate`、`NSIndexPath`这些类型，都支持Tagged Pointer技术。



##### 系统对tagged pointer的加密

在iOS12系统之前，发现是可以直接打印tagged pointer的值的，可读性非常好，但是12之后再打印就发现完全看不懂了。

```objectivec
- (void)testCase {
	NSString *stringWithFormat1 = [NSString stringWithFormat:@"y"];
    [self formatedLogObject:stringWithFormat1];
}

- (void)formatedLogObject:(id)object {
    if (@available(iOS 12.0, *)) {
        NSLog(@"%p %@ %@", object, object, object_getClass(object));
    } else {
        NSLog(@"0x%6lx %@ %@", object, object, object_getClass(object));
    }
}

复制代码
```

上面的测试代码，在12之前输出： 0x79是ASCII对应的y字符的值

```powershell
0xa000000000000791 y NSTaggedPointerString
```

iOS12之后输出：

```powershell
0xcb47b8d98a2fa15f y NSTaggedPointerString
```

iOS12之前打印指针的值能很清晰的看到数据等信息，iOS12之后系统则打印的完全看不懂了，看了源代码发现苹果是做了混淆，让我们不能直接得到值，从而避免我们去很容易就伪造出一个tagged pointer对象



## 内存管理

![img](https://sylarimage.oss-cn-shenzhen.aliyuncs.com/uPic/02ae03e4ec2a4d808bd1976b916b65d1~tplv-k3u1fbpfcp-zoom-1.png)



## 结论

1、Tagged Pointer有长度限制，过长会依然会采用对象的形式保存

2、Tagged Pointer没有isa指针，它不是一个对象，只是一个伪装成对象的普通变量而已。

3、Tagged Pointer是一个特殊的指针，不指向任何实质地址。 



## Reference

[1.iOS特有概念TaggedPointer](https://www.jianshu.com/p/408128f1dae3)

[2.OC内存管理-Tagged Pointer初探](https://juejin.im/post/6887543378628902925)

