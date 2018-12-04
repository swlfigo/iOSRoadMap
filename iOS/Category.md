### 12. Category

**extension在编译期决议**，它就是类的一部分，在编译期和头文件里的@interface以及实现文件里的@implement一起形成一个完整的类，它伴随类的产生而产生，亦随之一起消亡，extension一般用来隐藏类的私有信息，你必须有一个类的源码才能为一个类添加extension，所以你无法为系统的类比如NSString添加extension。

**category是在运行期决议的**。

category和extension的区别来看，我们可以推导出一个明显的事实，**extension可以添加实例变量**，而**category是无法添加实例变量的**（因为在运行期，对象的内存布局已经确定，如果添加实例变量就会破坏类的内部布局，这对编译型语言来说是灾难性的）【只能通过Runtime添加实例变量】

![Category](http://img.isylar.com/media/Category.png)

category的定义也可以看出category的可为（可以添加实例方法，类方法，甚至可以实现协议，添加属性）和不可为（无法添加实例变量）


## Category如何加载

首先拿到`category_t`数组 (获得所有`Category`文件)
1. 把category的实例方法、协议以及属性添加到类上
2. 把category的类方法和协议添加到类的metaclass上

把所有`category`的实例方法列表拼成了一个`大的实例方法列表`，然后转交给了attachMethodLists方法(方法，扩展也类似)

**需要注意的有两点：**
1)、category的方法**没有“完全替换掉”**原来类已经有的方法，也就是说如果category和原来类都有methodA，那么category附加完成之后，类的方法列表里会有两个methodA
2)、**category的方法被放到了新方法列表的前面，而原来类的方法被放到了新方法列表的后面**，这也就是我们平常所说的category的方法会“覆盖”掉原来类的同名方法，这是因为运行时在查找方法的时候是顺着方法列表的顺序查找的，它只要一找到对应名字的方法，就会罢休^_^，殊不知后面可能还有一样名字的方法。

## Category 和 +load

1)、在类的+load方法调用的时候，我们可以调用category中声明的方法么？

可以调用，因为附加category到类的工作会先于+load方法的执行(指"替换"原类方法)

2)、这么些个+load方法，调用顺序是咋样的呢？

+load的执行顺序是先类，后category，而category的+load执行顺序是根据编译顺序决定的。