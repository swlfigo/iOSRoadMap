# class_data_bits_t

由前面可知  [ObjectClass.md](./ObjectClass.md)



**objc_class**的真实定义实际的代码我们可以从 [objc-runtime-new.h](https://opensource.apple.com/source/objc4/objc4-750.1/runtime/objc-runtime-new.h.auto.html) 中看到(中间代码省略)：

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2020-04-23-143952.jpg)



```objective-c
struct class_data_bits_t {

    // Values are the FAST_ flags above.
    uintptr_t bits;
private:
    bool getBit(uintptr_t bit)
    {
        return bits & bit;
    }
  
  ...
}
```





 ObjC 中 `class_data_bits_t` 的结构体，其中只含有一个 64 位的 `bits` 用于存储与类有关的信息：



![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2020-05-19-141003.jpg)

`objc_class` 结构体中的注释写到 `class_data_bits_t` 相当于 `class_rw_t` 指针加上 rr/alloc 的标志。

```objective-c
class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags
```

为我们提供了便捷方法用于返回其中的 `class_rw_t *` 指针：

**objc_class** 中的 data 返回 `class_rw_t` 结构，此结构定义如下：

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2020-04-23-144757.png)

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2020-04-23-144612.jpg)



而`class_rw_t`是通过bits调用data方法得来的，来到data方法内部实现。我们可以看到，data函数内部仅仅对bits进行`&FAST_DATA_MASK`操作

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2020-04-23-144842.jpg)



将 `bits` 与 `FAST_DATA_MASK` 进行位运算，只取其中的 `[3, 47]` 位转换成 `class_rw_t *` 返回。

> 在 x86_64 架构上，Mac OS **只使用了其中的 47 位来为对象分配地址**。而且由于地址要按字节在内存中按字节对齐，所以掩码的后三位都是 0。

因为 `class_rw_t *` 指针只存于第 `[3, 47]` 位，所以可以使用最后三位来存储关于当前类的其他信息：

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2020-05-19-144111.jpg)

```objective-c
#define FAST_IS_SWIFT           (1UL<<0)
#define FAST_HAS_DEFAULT_RR     (1UL<<1)
#define FAST_REQUIRES_RAW_ISA   (1UL<<2)
#define FAST_DATA_MASK          0x00007ffffffffff8UL
```

- ```
  isSwift()
  ```

  - `FAST_IS_SWIFT` 用于判断 Swift 类

- ```
  hasDefaultRR()
  ```

  - `FAST_HAS_DEFAULT_RR` 当前类或者父类含有默认的 `retain/release/autorelease/retainCount/_tryRetain/_isDeallocating/retainWeakReference/allowsWeakReference` 方法

- ```
  requiresRawIsa()
  ```

  - `FAST_REQUIRES_RAW_ISA` 当前类的实例需要 raw `isa`



所以调用初始化如下

```objective-c
// objc_class 中的 data() 方法
class_data_bits_t bits;

class_rw_t *data() {
   return bits.data();
}

// class_data_bits_t 中的 data() 方法
uintptr_t bits;

class_rw_t* data() {
   return (class_rw_t *)(bits & FAST_DATA_MASK);
}
```

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2020-04-23-144612.jpg)

而成员变量信息则是存储在`class_ro_t`内部中的，我们来到`class_ro_t`内查看。
`class_rw_t` 表示**read write**，`class_ro_t` 表示 **read only**。

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2020-04-23-145011.jpg)



### `class_rw_t` 和 `class_ro_t`

ObjC 类中的属性、方法还有遵循的协议等信息都保存在 `class_rw_t` 中：

```objective-c
struct class_rw_t {
    uint32_t flags;
    uint32_t version;

    const class_ro_t *ro;

    method_array_t methods;
    property_array_t properties;
    protocol_array_t protocols;

    Class firstSubclass;
    Class nextSiblingClass;
};
```



其中还有一个指向常量的指针 `ro`，其中存储了**当前类在编译期就已经确定的属性、方法以及遵循的协议**。(如果是当前类有Category扩展，则新增的属性方法会放在 `class_rw_t` 的 `methods` 、`properties` 数组中，成为一个二维数组)



```objective-c
struct class_ro_t {
    uint32_t flags;
    uint32_t instanceStart;
    uint32_t instanceSize;
    uint32_t reserved;

    const uint8_t * ivarLayout;

    const char * name;
    method_list_t * baseMethodList;
    protocol_list_t * baseProtocols;
    const ivar_list_t * ivars;

    const uint8_t * weakIvarLayout;
    property_list_t *baseProperties;
};
```



**在编译期间**类的结构中的 `class_data_bits_t *data` 指向的是一个 `class_ro_t *` 指针：

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2020-07-30-140953.jpg)

然后在加载 *ObjC 运行时*的过程中在 `realizeClass` 方法中：

1. 从 `class_data_bits_t` 调用 `data` 方法，将结果从 `class_rw_t` 强制转换为 `class_ro_t` 指针
2. 初始化一个 `class_rw_t` 结构体
3. 设置结构体 `ro` 的值以及 `flag`
4. 最后设置正确的 `data`。

```objectivec
const class_ro_t *ro = (const class_ro_t *)cls->data();
class_rw_t *rw = (class_rw_t *)calloc(sizeof(class_rw_t), 1);
rw->ro = ro;
rw->flags = RW_REALIZED|RW_REALIZING;
cls->setData(rw);
```

下图是 `realizeClass` 方法执行过后的类所占用内存的布局

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2020-07-30-141124.jpg)

但是，在这段代码运行之后 `class_rw_t` 中的方法，属性以及协议列表均为空。这时需要 `realizeClass` 调用 `methodizeClass` 方法来**将类自己实现的方法（包括分类）、属性和遵循的协议加载到 `methods`、 `properties` 和 `protocols` 列表中**。



## 小结

1. 在内存中的位置是在编译期间决定的，在之后修改代码，也不会改变内存中的位置。
2. 类的方法、属性以及协议在编译期间存放到了“错误”的位置，直到 `realizeClass` 执行之后，才放到了 `class_rw_t` 指向的只读区域 `class_ro_t`，这样我们即可以在运行时为 `class_rw_t` 添加方法，也不会影响类的只读结构。
3. 在 `class_ro_t` 中的属性在运行期间就不能改变了，再添加方法时，会修改 `class_rw_t` 中的 `methods` 列表，而不是 `class_ro_t` 中的 `baseMethods`，

## Reference

[深入解析 ObjC 中方法的结构](https://draveness.me/method-struct/)

