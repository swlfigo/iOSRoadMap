### 4. Autoreleasepool 有什么作用 , 如何实现 , 什么时候释放

AutoreleasePool（自动释放池）是OC中的一种内存自动回收机制，它可以延迟加入AutoreleasePool中的变量release的时机。

在没有手加Autorelease Pool的情况下， **Autorelease对象是在当前的runloop迭代结束时释放的，而它能够释放的原因是系统在每个runloop迭代中都加入了自动释放池Push和Pop**

AutoreleasePool的内存结构就是一个双向链表。一个线程的autoreleasepool就是一个指针栈。
栈中存放的指针指向加入需要release的对象或者POOL_SENTINEL（哨兵对象，用于分隔autoreleasepool）。
栈中指向POOL_SENTINEL的指针就是autoreleasepool的一个标记。当autoreleasepool进行出栈操作，每一个比这个哨兵对象后进栈的对象都会release。
这个栈是由一个以page为节点双向链表组成，page根据需求进行增减。
autoreleasepool对应的线程存储了指向最新page（也就是最新添加autorelease对象的page）的指针。