# HTTP
超文本传输协议

## HTTP的请求方式有哪些
`GET` `POST` `PUT` `DELETE` `HEAD` `OPTIONS`

## GET和POST方式的区别
* GET请求参数以?分割拼接到URL后面,POST请求参数在Body里面
* GET参数长度限制2048个字符,POST一般没有限制
* GET:获取资源
    *安全的，幂等的，可缓存的
* POST:处理资源
    *非安全的，非幂等的，不可缓存的

### 安全性
不应该引起Server端的任何状态变化: `GET`,`HEAD`,`OPTIONS`

### 幂等性
同一个请求方法执行多次和执行一次的效果完全相同

### 可缓存性
请求是否可以被缓存

## 状态码
1xx,2xx,3xx,4xx,5xx

## HTTP特点
无连接(HTTP有建立连接和释放连接过程),解决这个缺点，可以通过 `HTTP的持久连接`解决
无状态(需要使用cookie/session解决)

### 持久连接
![](media/15499388912503.jpg)

一段时间后才关闭,采用`持久连接方案`,实际上是提升了网络请求的效率.

### 头部字段
* Connection:keep-alive 
* time:20 (持久连接多少时间有效)
* max:10 (指这条连接最多可以发生多少次Http请求)

### 怎样判断一个请求是否结束
* Content-length:1024
* chunked,最后会有一个空的chunked