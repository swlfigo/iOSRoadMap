# AssociatedObject关联对象的内部实现



#### AssociatedObject关联对象

**为什么要引入关联对象？**

> - 一般我们需要对现有的类做扩展，可以通过继承、类别等方式去实现；当我们使用类别的方式扩展，如果对现有的类增加属性的话，编译器是不会生成实例变量；类别的结构体中没有ivar的结构体，同时类的ivar设计的是一个const
> - 类别是运行时装载到类中的，当类realizeClass之后它的`instanceSize`就已经确定无法修改了，这些操作都是在load之前，main函数之前
> - 如果想通过runtime的方法class_addIvar它只适用于新建一个类的时候增加，对于类别中增加实例就不适用
> - 关联对象就是在不改变类的结构的情况下，将类需要关联的对象存储在关联表中，那么类别中添加的属性的值的存取就可以通过关联来解决



##### 主要函数



```objective-c
// 设置关联对象函数
OBJC_EXPORT void
objc_setAssociatedObject(id _Nonnull object, const void * _Nonnull key,
                         id _Nullable value, objc_AssociationPolicy policy)
    OBJC_AVAILABLE(10.6, 3.1, 9.0, 1.0, 2.0);
// 获取关联的对象函数
OBJC_EXPORT id _Nullable
objc_getAssociatedObject(id _Nonnull object, const void * _Nonnull key)
    OBJC_AVAILABLE(10.6, 3.1, 9.0, 1.0, 2.0);
// 删除对象的关联对象函数
OBJC_EXPORT void
objc_removeAssociatedObjects(id _Nonnull object)
    OBJC_AVAILABLE(10.6, 3.1, 9.0, 1.0, 2.0);
```

- objc_setAssociatedObject -- 关联对象value到object
   `object`：宿主对象
   `key`：关联对象的key，一般传入一个常量的地址作为唯一标识
   `value`：被关联的对象
   `policy`：关联的规则，主要是内存管理的规则
- objc_getAssociatedObject -- 从object中根据key获取关联的对象的value
   `object`：宿主对象
   `key`：关联对象的key，传入设置时候传入的key
- objc_removeAssociatedObjects -- 删除object的所有的关联的对象
   `object`：宿主对象



实现关联对象技术的核心对象有：

- `AssociationsManager`
- `AssociationsHashMap`
- `ObjectAssociationMap`
- `ObjcAssociation`





###### 源码 （下面以set方法为例 get类似



```cpp
void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy) {
    _object_set_associative_reference(object, (void *)key, value, policy);
}
```





```php
void _object_set_associative_reference(id object, void *key, id value, uintptr_t policy) {
    // retain the new value (if any) outside the lock.
    ObjcAssociation old_association(0, nil);
    id new_value = value ? acquireValue(value, policy) : nil;
    {
        AssociationsManager manager;
        AssociationsHashMap &associations(manager.associations());
        disguised_ptr_t disguised_object = DISGUISE(object);
        if (new_value) {//如果set方法传值不是nil
            // break any existing association.
            AssociationsHashMap::iterator i = associations.find(disguised_object);
            if (i != associations.end()) {
                //如果AssociationsHashMap已经存在 进行下一步
                // secondary table exists
                ObjectAssociationMap *refs = i->second;
                ObjectAssociationMap::iterator j = refs->find(key);
                if (j != refs->end()) {
                    //更改值
                    old_association = j->second;
                    j->second = ObjcAssociation(policy, new_value);
                } else {
                    //添加新值
                    (*refs)[key] = ObjcAssociation(policy, new_value);
                }
            } else {
                 //AssociationsHashMap不存在 创建 并 添加
                // create the new association (first time).
                ObjectAssociationMap *refs = new ObjectAssociationMap;
                associations[disguised_object] = refs;
                (*refs)[key] = ObjcAssociation(policy, new_value);
                object->setHasAssociatedObjects();
            }
        } else {
            //如果set方法传值是nil
            // setting the association to nil breaks the association.
            AssociationsHashMap::iterator i = associations.find(disguised_object);
            if (i !=  associations.end()) {
                ObjectAssociationMap *refs = i->second;
                ObjectAssociationMap::iterator j = refs->find(key);
                if (j != refs->end()) {
                    old_association = j->second;
                    refs->erase(j);//擦除
                }
            }
        }
    }
    // release the old value (outside of the lock).
    if (old_association.hasValue()) ReleaseValue()(old_association);
}
```



![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-01-25-135848.jpg)



![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-01-25-135915.jpg)

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-01-25-135933.jpg)



![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-01-25-135948.jpg)



![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-01-25-135956.jpg)

可以得出结论：

- 关联对象并**不是存储在被关联对象本身内存中**
- 关联对象由 `AssociationsManager` 管理并在 `AssiciationsHashMap` 存储。
- 所有对象的关联内容都在同一个全局容器中。
- 设置关联对象为nil，就相当于是移除关联对象



![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-14-095543.jpg)



- 关联对象的释放时机与移除时机并不总是一致，比如实验中用关联策略 **OBJC_ASSOCIATION_ASSIGN** 进行**关联的对象**，很早就已经被释放了，但是并没有被移除，而再使用这个关联对象时就会造成 Crash 。[注意是用Assign关联对象(@property中用assign也会导致崩溃)]

###  

## Reference

[1.探索AssociatedObject关联对象的内部实现](https://www.jianshu.com/p/c109015e8c9a)

[2.Objective - C 关联对象(二) 关联对象的底层数据结构](https://www.jianshu.com/p/9a8502938dab)