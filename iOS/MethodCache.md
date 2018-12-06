#  15. Method Cache

objc_msgSend（就arm平台而言）的消息分发分为以下几个步骤：

* 判断receiver是否为nil，也就是objc_msgSend的第一个参数self，也就是要调用的那个方法所属对象

* 从缓存里寻找，找到了则分发，否则

* 利用 `objc-class.mm` 中_class_lookupMethodAndLoadCache3 方法去寻找selector

     * 如果支持GC，忽略掉非GC环境的方法（retain等）
     * 从 **本class的method list寻找selector**，如果找到，填充到缓存中，并返回selector，否则
     * **寻找父类的method list，并依次往上寻找，直到找到selector**，填充到缓存中，并返回selector，否则
     * 调用_class_resolveMethod，如果可以动态resolve为一个selector，不缓存，方法返回，否则
     * 转发这个selector，否则
* 报错，抛出异常

**缓存的存储使用了散列表(HashTable)。因为散列表检索起来更快，**

## 1. 方法缓存存在什么地方？
在类的定义里就有cache字段，没错，类的所有缓存都存在metaclass上，所以每个类都只有一份方法缓存，而**不是每一个类的object都保存一份**

```
  struct _class_t {
      struct _class_t *isa;
      struct _class_t *superclass;
      void *cache;  //缓存
      void *vtable;
      struct _class_ro_t *ro;
  };
```

## 2. 父类方法的缓存只存在父类么，还是子类也会缓存父类的方法？

即便是从父类取到的方法，**也会存在类本身的方法缓存里**。而当用一个父类对象去调用那个方法的时候，也会在父类的metaclass里缓存一份。

## 3. 为什么 类的方法列表 不直接做成散列表呢，做成list，还要单独缓存，多费事？

* 散列表是没有顺序的，Objective-C的方法列表是一个list，是有顺序的；Objective-C在查找方法的时候会顺着list依次寻找，并且category的方法在原始方法list的前面，需要先被找到，如果直接用hash存方法，方法的顺序就没法保证。

* list的方法还保存了除了selector和imp之外其他很多属性

* 散列表是有空槽的，会浪费空间