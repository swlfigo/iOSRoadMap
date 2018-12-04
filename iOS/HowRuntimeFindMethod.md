### 6.Runtime 是如何找到实例方法的具体实现的？

一个OC方法被编译成objc_msgSend，因为OC中存在一种对象叫做类对象（Class Object），类对象中保存了方法的列表，superClass等信息。objc_msgSend这个C函数输入参数包括，id类型的self，SEL类型的_cmd,以及参数列表。很直观，id类型中一定存在一个可以找到类对象的指针。 

* OC的对象通过isa找到类对象
* 类对象查找自己存储的方法列表来找到对应的方法执行体
* 方法执行体执行具体的代码，并返回值给调用者。

我们来看一个例子， 
定义一个

```objectivec
@interface CustomObject : NSObject
-(NSString *)returnMeHelloWorld;
@end
@implementation CustomObject

-(NSString *)returnMeHelloWorld{
    return @"hello world";
}
@end
```

我们先只看调用这一行

```objectivec
    NSString * helloWorld =  [obj returnMeHelloWorld];
```

* 编译成如下id objc_msgSend(self,@selector(returnMeHelloWorld:));
* 在self中沿着isa找到CustomObject的类对象
* 类对象查找自己的方法list，找到对应的方法执行体Method
* 把参数传递给IMP实际指向的执行代码
* 代码执行返回结果给helloWorld
