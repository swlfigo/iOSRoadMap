# 18 KVC

![KV](http://img.isylar.com/media/KVC.png)

## Accessor Method
如果实现了下面的相似的方法名称

*  getKey
*  key
*  isKey

`valueForKey` 调用方法，则判断为存在对应的访问器方法

## Instance var
如果存在下面的成员变量
* _key
* _isKey
* key
* isKey

也能获得相应的变量

