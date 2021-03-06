# iOS 界面渲染流程

![15420320733034](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-07-23-134948.jpg)


1. 首先一个视图由 CPU 进行 Frame 布局，准备视图和图层的层级关系，查询是否有重写 `drawRect:` 或 `drawLayer:inContext: `方法，**注意：如果有重写的话，这里的渲染是会占用CPU进行处理的。**

2. CPU 会将处理视图和图层的层级关系打包，通过 IPC（内部处理通信）通道提交给渲染服务，渲染服务由 OpenGL ES 和 GPU 组成。

3. 渲染服务首先将图层数据交给 OpenGL ES 进行纹理生成和着色。生成前后帧缓存，再根据显示硬件的刷新频率，一般以设备的Vsync信号和CADisplayLink为标准，进行前后帧缓存的切换。

4. 最后，将最终要显示在画面上的后帧缓存交给 GPU，进行采集图片和形状，运行变换，应用文理和混合。最终显示在屏幕上。


> 在iOS中是双缓冲机制，有前帧缓存、后帧缓存，即GPU会预先渲染好一帧放入一个缓冲区内（前帧缓存），让视频控制器读取，当下一帧渲染好后，GPU会直接把视频控制器的指针指向第二个缓冲器（后帧缓存）。当你视频控制器已经读完一帧，准备读下一帧的时候，GPU会等待显示器的VSync信号发出后，前帧缓存和后帧缓存会瞬间切换，后帧缓存会变成新的前帧缓存，同时旧的前帧缓存会变成新的后帧缓存。

