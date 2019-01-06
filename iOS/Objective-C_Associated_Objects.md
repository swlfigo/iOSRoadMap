### 10. Objective-C Associated Objects 的实现原理

## 关联对象的本质
1. 关联对象由 `AssociationsManager` 管理并在 `AssociationsHashMap` 存储
2. 所有对象的关联内容都在 `同一个全局容器` 中

![AssociationsHashMap](http://img.isylar.com/media/AssociationsHashMap.png)



## Objective-C_Associated_Objects
1. 关联对象与被关联对象本身的存储并没有直接的关系，它是存储在单独的哈希表中的；
2. 关联对象的五种关联策略与属性的限定符非常类似，在绝大多数情况下，我们都会使用 **OBJC_ASSOCIATION_RETAIN_NONATOMIC** 的关联策略，这可以保证我们持有关联对象；
3. 关联对象的释放时机与移除时机并不总是一致，比如实验中用关联策略 **OBJC_ASSOCIATION_ASSIGN** 进行关联的对象，很早就已经被释放了，但是并没有被移除，而再使用这个关联对象时就会造成 Crash 。

