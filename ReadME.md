# iOS RoadMap
![buildAction](https://img.shields.io/github/workflow/status/swlfigo/iOSRoadMap/Gitbook%20Build)

#### 更多博文可看传送门 ↓

[传送门](http://isylar.com)

## iOS

[iOS](/iOS/ReadME.md)



## 架构方面

### 1. 设计模式

#### MVC

##### 什么是MVC？

MVC最早存在于桌面程序中的, M是指业务数据, V是指用户界面, C则是控制器. 在具体的业务场景中, C作为M和V之间的连接, 负责获取输入的业务数据, 然后将处理后的数据输出到界面上做相应展示, 另外, 在数据有所更新时, C还需要及时提交相应更新到界面展示. 在上述过程中, 因为M和V之间是完全隔离的, 所以在业务场景切换时, 通常只需要替换相应的C, 复用已有的M和V便可快速搭建新的业务场景. MVC因其复用性, 大大提高了开发效率, 现已被广泛应用在各端开发中。

[2.设计模式](knowledge/DesignPatterns.md)

## 网络
[1. 简述TCP的三次握手过程](network/TCP-Three-Way-Handshake.md)

[2. 4次挥手过程详解](network/4次挥手过程详解.md)

[3. TCP/UDP区别以及UDP如何实现可靠传输](network/TCP:UDP区别以及UDP如何实现可靠传输.md)

[4. Http 和 Https 有什么关系和区别](network/Http%20和%20Https%20有什么关系和区别.md)

[5. get 和 post 区别](network/get%20和%20post%20区别.md)

[6. 什么是Http协议无状态协议?怎么解决Http协议无状态协议?](network/什么是Http协议无状态协议%3F怎么解决Http协议无状态协议%3F.md)

[7. 一次完整的HTTP请求所经历的7个步骤](network/一次完整的HTTP请求所经历的7个步骤.md)

[8. Socket](./network/Socket.md)

[9. Socket & Http](./network/Http&Socket.md)

[10.OSI](./network/OSI.md)

## 一些推荐阅读

1. [《图解HTTP》知识点摘录](https://juejin.im/post/5aa62f93f265da23906ba830)
2. [iOS 消息发送与转发详解](https://juejin.im/post/5aa79411f265da237a4cb045)



-------

## 杂乱知识点

 1.App启动过程 - [链接](knowledge/App启动.md)

2.Cocoapods原理总结 - [链接](https://juejin.im/entry/59dd94b06fb9a0451463030b)

3.内联函数,与宏的区别 - [链接](https://github.com/swlfigo/iOSInterview/blob/master/%E6%9D%82%E4%B9%B1%E7%9F%A5%E8%AF%86%E7%82%B9/static_inline.md)

4.[单链表与顺序结构](/knowledge/listAndNode.md)

5.[static区别](/knowledge/staticCompare.md)