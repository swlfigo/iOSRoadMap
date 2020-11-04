# OC对象创建过程

## 创建对象的两种方法

### [[Class alloc] init]



```objective-c
+ (id)alloc {    
  return _objc_rootAlloc(self);
}
// Base class implementation of +alloc. cls is not nil.
// Calls [cls allocWithZone:nil].
id_objc_rootAlloc(Class cls){    
  return callAlloc(cls, false/*checkNil*/, true/*allocWithZone*/);
}
```



```objective-c
// Replaced by CF (throws an NSException)
+ (id)init {    
  return (id)self;
}

- (id)init {    
  return _objc_rootInit(self);
}

id_objc_rootInit(id obj){    
  // In practice, it will be hard to rely on this function.    
  // Many classes do not properly chain -init calls.    
  return obj;
}
```



### [Class new]



```objective-c
+ (id)new {    
  return [callAlloc(self, false/*checkNil*/) init];
}

- (id)init {    
  return _objc_rootInit(self);
}
```



从上面两种创建对象的方法可以看出第一种方式对象的创建是在alloc中，init方法只是返回已经创建的对象。通过new方法创建的对象本质还是alloc和init的结合。



## callAlloc

```objective-c

// Call [cls alloc] or [cls allocWithZone:nil], with appropriate 

// shortcutting optimizations.
static ALWAYS_INLINE id
callAlloc(Class cls, bool checkNil, bool allocWithZone=false)
{
    if (slowpath(checkNil && !cls)) return nil;

#if __OBJC2__
    if (fastpath(!cls->ISA()->hasCustomAWZ())) {
        // No alloc/allocWithZone implementation. Go straight to the allocator.
        // fixme store hasCustomAWZ in the non-meta class and 
        // add it to canAllocFast's summary
        if (fastpath(cls->canAllocFast())) {
            // No ctors, raw isa, etc. Go straight to the metal.
            bool dtor = cls->hasCxxDtor();
            id obj = (id)calloc(1, cls->bits.fastInstanceSize());
            if (slowpath(!obj)) return callBadAllocHandler(cls);
            obj->initInstanceIsa(cls, dtor);
            return obj;
        }
        else {
            // Has ctor or raw isa or something. Use the slower path.
            id obj = class_createInstance(cls, 0);
            if (slowpath(!obj)) return callBadAllocHandler(cls);
            return obj;
        }
    }
#endif
    // No shortcuts available.
    if (allocWithZone) return [cls allocWithZone:nil];
    return [cls alloc];
}
```





## Reference

[1.带你深入了解OC对象创建过程](https://mp.weixin.qq.com/s/cyNCgBNO9nigvfzDpjzR2g)