# 9 属性关键字

### 读写权限
* readonly
* `readwrite(Default)`

### 原子性
* `atomic(Default)`
* nonatomic

`atomic` 保证赋值获取是线程安全,是对成员属性的直接的获取安全，并不代表操作和访问.
比如 `atomic` 修饰的是一个数组,对数组赋值获取是安全的，但是对数组进行操作(添加对象，移除对象)是不保证线程不安全的.

### 引用计算

* retain/strong
* assign/unsafe_unretained
* weak
* copy

`assgin`特点:
* 修饰基本数据类型，如`int`,`BOOL`等
* 修饰对象类型时,不改变其引用计数.
* 会产生悬垂指针 (`assign`修饰的对象，被释放后，`assign`指针还是指向原对象地址，继续访问原对象的话可能会蹦)

`weak`特点
* 不改变被修饰对象的引用计数，多用于解决循环引用问题
* 所指的对象在被释放后会自动置为nil


## `assign` vs `weak`:

* `weak` 可以修饰对象，`assgin`可以修饰对象与基本数据类型.
* `assign`修饰的对象，被释放后，`assign`指针还是指向原对象地址，继续访问原对象的话可能会蹦，`weak` 所指的对象在被释放后会自动置为nil

`weak` 此特质表明该属性定义了一种 “非拥有关系” (nonowning relationship)。为这种属性设置新值时，设置方法既不保留新值，也不释放旧值。此特质同assign类似， 然而在属性所指的对象遭到摧毁时，属性值也会清空(nil out)。 而 assign 的“设置方法”只会执行针对“纯量类型” (scalar type，例如 CGFloat 或 NSlnteger 等)的简单赋值操作。


## Copy

* 可变对象的copy和mutableCopy都是深拷贝
* 不可变对象的copy是浅拷贝，mutableCopy是深拷贝
* copy方法返回的都是不可变对象
 
![](http://img.isylar.com/media/15467745263315.jpg)
