#  19. Runtime

## objc_object && objc_class

![objc_object](http://img.isylar.com/media/objc_object.png)

`id` 类型对象对应 `runtime` 中为 `objc_object` 结构的对象
`Class` 对象对应 `runtime` 中为 `objc_class`
`objec_class` 继承于 `objc_object`

## Rumtime数据结构

### isa

`isa` 数据结构为一个c++中 共用体结构, objc中定义为 `isa_t` 名称,由 32位或者64位(看系统结构) 0或1 组成

![](http://img.isylar.com/media/15473493708247.jpg)

`isa` 分 `指针型isa` 与非 `指针型isa`

#### 指针型isa
`isa` 64位的 `值` 代表 `Class` 的地址 
#### 非指针isa
`isa` 的 `部分值`  代表 `Class` 的地址(作用是不一定需要这么多位数的值可以表达出地址，不用浪费过多位置，剩余部分还能存储其他数据)

#### isa指向

* 关于对象,其指向类对象
* 关于类对象,其指向元类对象

### cache_t
* 用于快速查找方法执行函数
* 是可增量的扩展的哈希表结构
* 是局部性原理的最佳应用

![](http://img.isylar.com/media/15473501512279.jpg)

### class_data_bits_t
* `class_data_bits_t` 主要是对 `class_rw_t` 的封装
* `class_rw_t` 代表了类相关的读写信息，是对 `class_ro_t`
* `class_ro_t` 代表了类相关的只读信息

#### class_rw_t

![class_rw_t](http://img.isylar.com/media/class_rw_t-1.png)

### method_t

![](http://img.isylar.com/media/15473530130138.jpg)

#### Type Encodings
* const char* types

| 返回值 | 参数1 | 参数2 | ... | 参数n |
| --- | --- | :-- | :-- | :-- |

```
-(void)aMethod   =>   v@:  (v代表void,@参数1,:参数2)
```

| V | @ | : |
| --- | --- | --- |
| void | id | SEL |

### 整体结构

![](http://img.isylar.com/media/15473538050738.jpg)


## 对象、类对象、元类对象
* 类对象存储实例方法列表等信息
* 元类对象存储类方法列表等信息

## 消息传递

如笔试题:

```objectivec
NSLog(NSStringFromClass([self Class]));
NSLog(NSStringFromClass([super Class]));
```

```objectivec
void objc_msgSend(void /* id self,SEL op,...*/)

//编译器转化为
[self class] <=> objc_msgSend(self,@selector(class))
```

```objectivec
void objc_msgSendSuper(void /*struct objc_super *super , SEL op,...*/)

struct objc_super{
    __unsafe_unretained id receiver;
}

[super class] <=> objc_msgSendSuper(super,@selector(class))

```
![RuntimeMethod](http://img.isylar.com/media/RuntimeMethod.png)

其消息接受者总为`self`,因为 `self` 没有 `class` 方法实现，会根据 `isa`指针逐级父类向上查找，一直找到根类 `NSObject`,找到并打印. `super` 的接受者是 `self`, 意思是从 `self` 的父类 `类的类对象方法`开始查找，跨越了当前类的`类对象方法列表`查找.

### 缓存查找
给定值是 SEL , 目标值是对应 bucket_t 中的 IMP

![](http://img.isylar.com/media/15473625906583.jpg)

### 当前类中查找
* 对于 已排好序 的列表,采用 `二分法查找` 算法查找方法对应的执行函数
* 对于 没有排序 的列表,采用 `一般遍历` 查找方法对应执行函数

### 消息转发机制



