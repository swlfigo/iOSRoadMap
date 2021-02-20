# KVC

是一种键值对设计模式，破坏面对对象的编程思想。(不重写特定方法,找不到Key情况下会崩溃)

主要方法

```objective-c
-(id)valueForKey:(NSString *)key
-(void)setValue:(id)value forked:(NSString *)key;
```



寻找路径

`setterKey(keySet方法)` -> `_key` -> `_isKey` -> `key` -> `iskey`

![](https://sylarimage.oss-cn-shenzhen.aliyuncs.com/20190318150641.png)

## 

**KVC setvalue:forkey与setvalue:forkeypath的区别**:

`forkey`用于简单路径,`forkeypath`用于复合路径(比如key是对象，可以直接赋值给这个对象的属性.eg:`setValue:@100 forKeyPath:@"person.number"`)