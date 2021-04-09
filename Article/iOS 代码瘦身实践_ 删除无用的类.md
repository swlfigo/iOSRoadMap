> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/6844903921169727496)

本文将提供一种静态分析的方式，用于查找可执行文件 Mach-o 中未使用的类，源码链接:[xuezhulian/classunref](https://github.com/xuezhulian/classunref)。

`Mach-o`文件中`__DATA __objc_classrefs`段记录了引用类的地址，`__DATA __objc_classlist`段记录了所有类的地址，取差集可以得到未使用的类的地址，然后进行符号化，就可以得到未被引用的类信息。

### 引用类地址

可以通过 Mac 自带的工具`otool`打印`Mach-o`中的段信息，需要注意的是模拟器和真机对应的可执行文件，数据的存储方式不同需要加以区分。 可以通过`file`命令获取到`arch`。

```
#binary_file_arch: distinguish Big-Endian and Little-Endian
#file -b output example: Mach-O 64-bit executable arm64
binary_file_arch = os.popen('file -b ' + path).read().split(' ')[-1].strip()
复制代码
```

在取类地址的时候区分`x86_64`和`arm`。

```
def pointers_from_binary(line, binary_file_arch):
    line = line[16:].strip().split(' ')
    pointers = set()
    if binary_file_arch == 'x86_64':
        #untreated line example:00000001030cec80	d8 75 15 03 01 00 00 00 68 77 15 03 01 00 00 00
        pointers.add(''.join(line[4:8][::-1] + line[0:4][::-1]))
        pointers.add(''.join(line[12:16][::-1] + line[8:12][::-1]))
        return pointers
    #arm64 confirmed,armv7 arm7s unconfirmed
    if binary_file_arch.startswith('arm'):
        #untreated line example:00000001030bcd20	03138580 00000001 03138878 00000001
        pointers.add(line[1] + line[0])
        pointers.add(line[3] + line[2])
        return pointers
    return None
复制代码
```

通过`otool -v -s __DATA __objc_classrefs`获取到引用类的地址。

```
def class_ref_pointers(path, binary_file_arch):
    ref_pointers = set()
    lines = os.popen('/usr/bin/otool -v -s __DATA __objc_classrefs %s' % path).readlines()
    for line in lines:
        pointers = pointers_from_binary(line, binary_file_arch)
        ref_pointers = ref_pointers.union(pointers)
    return ref_pointers
复制代码
```

### 所有类地址

通过`otool -v -s __DATA __objc_classlist`获取所有类的地址。

```
def class_list_pointers(path, binary_file_arch):
    list_pointers = set()
    lines = os.popen('/usr/bin/otool -v -s __DATA __objc_classlist %s' % path).readlines()
    for line in lines:
        pointers = pointers_from_binary(line, binary_file_arch)
        list_pointers = list_pointers.union(pointers)
    return list_pointers
复制代码
```

### 取差集

用所有类信息减去引用类的信息，此时我们可以拿到未使用类的地址信息。

```
unref_pointers = class_list_pointers(path, binary_file_arch) - class_ref_pointers(path, binary_file_arch)
复制代码
```

### 符号化

通过`nm -nm`命令可以得到地址和对应的类名字。

```
def class_symbols(path):
    symbols = {}
    #class symbol format from nm: 0000000103113f68 (__DATA,__objc_data) external _OBJC_CLASS_$_EpisodeStatusDetailItemView
    re_class_name = re.compile('(\w{16}) .* _OBJC_CLASS_\$_(.+)')
    lines = os.popen('nm -nm %s' % path).readlines()
    for line in lines:
        result = re_class_name.findall(line)
        if result:
            (address, symbol) = result[0]
            symbols[address] = symbol
    return symbols
复制代码
```

### 过滤

在实际分析的过程中发现，如果一个类的子类被实例化，父类未被实例化，此时父类不会出现在`__objc_classrefs`这个段里，在未使用的类中需要将这一部分父类过滤出去。使用`otool -oV`可以获取到类的继承关系。

```
def filter_super_class(unref_symbols):
    re_subclass_name = re.compile("\w{16} 0x\w{9} _OBJC_CLASS_\$_(.+)")
    re_superclass_name = re.compile("\s*superclass 0x\w{9} _OBJC_CLASS_\$_(.+)")
    #subclass example: 0000000102bd8070 0x103113f68 _OBJC_CLASS_$_TTEpisodeStatusDetailItemView
    #superclass example: superclass 0x10313bb80 _OBJC_CLASS_$_TTBaseControl
    lines = os.popen("/usr/bin/otool -oV %s" % path).readlines()
    subclass_name = ""
    superclass_name = ""
    for line in lines:
        subclass_match_result = re_subclass_name.findall(line)
        if subclass_match_result:
            subclass_name = subclass_match_result[0]
        superclass_match_result = re_superclass_name.findall(line)
        if superclass_match_result:
            superclass_name = superclass_match_result[0]

        if len(subclass_name) > 0 and len(superclass_name) > 0:
            if superclass_name in unref_symbols and subclass_name not in unref_symbols:
                unref_symbols.remove(superclass_name)
            superclass_name = ""
            subclass_name = ""
    return unref_symbols
复制代码
```

为了防止一些三方库的误伤，还可以去过滤一些前缀，或者是是仅保留带有某些前缀的类。

```
for unref_pointer in unref_pointers:
        if unref_pointer in symbols:
            unref_symbol = symbols[unref_pointer]
            if len(reserved_prefix) > 0 and not unref_symbol.startswith(reserved_prefix):
                continue
            if len(filter_prefix) > 0 and unref_symbol.startswith(filter_prefix):
                continue
            unref_symbols.add(unref_symbol)
复制代码
```

最终结果保存在脚本目录下。

```
script_path = sys.path[0].strip()
f = open(script_path+"/result.txt","w")
f.write( "unref class number:   %d\n" % len(unref_symbles))
f.write("\n")
for unref_symble in unref_symbles:
    f.write(unref_symble+"\n")
f.close()
复制代码
```

这个思路在一定程度上能够减少代码的冗余，减小包的体积。因为是静态分析，不能包括动态调用的情况，对于需要删除的类需要进一步的确认。