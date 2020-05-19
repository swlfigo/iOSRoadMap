# class_data_bits_t

由前面可知

**objc_class**的真实定义实际的代码我们可以从 [objc-runtime-new.h](https://opensource.apple.com/source/objc4/objc4-750.1/runtime/objc-runtime-new.h.auto.html) 中看到(中间代码省略)：

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2020-04-23-143952.jpg)

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







而成员变量信息则是存储在`class_ro_t`内部中的，我们来到`class_ro_t`内查看。
`class_rw_t` 表示**read write**，`class_ro_t` 表示 **read only**。

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2020-04-23-145011.jpg)



## 