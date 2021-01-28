#   Block

## 1.什么是Block与Block构造

> In programming languages, a closure is a function or reference to a function together with a referencing environment—a table storing a reference to each of the non-local variables (also called free variables or upvalues) of that function.

* `Block` 是将函数及其执行上下文封装起来的`对象`
* `Block`的调用即是函数的调用

如：

```objective-c
int multiplier = 6
int(^Block)(int) = ^int(int num){
    return num * multiplier;
}
```

