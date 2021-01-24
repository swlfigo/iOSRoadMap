# AssociatedObject关联对象的内部实现



#### AssociatedObject关联对象

**为什么要引入关联对象？**

> - 一般我们需要对现有的类做扩展，可以通过继承、类别等方式去实现；当我们使用类别的方式扩展，如果对现有的类增加属性的话，编译器是不会生成实例变量；类别的结构体中没有ivar的结构体，同时类的ivar设计的是一个const
> - 类别是运行时装载到类中的，当类realizeClass之后它的`instanceSize`就已经确定无法修改了，这些操作都是在load之前，main函数之前
> - 如果想通过runtime的方法class_addIvar它只适用于新建一个类的时候增加，对于类别中增加实例就不适用
> - 关联对象就是在不改变类的结构的情况下，将类需要关联的对象存储在关联表中，那么类别中添加的属性的值的存取就可以通过关联来解决





## Reference

[1.探索AssociatedObject关联对象的内部实现](https://www.jianshu.com/p/c109015e8c9a)

