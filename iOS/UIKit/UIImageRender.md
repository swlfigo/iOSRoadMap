# iOS 图片的加载与渲染过程

1. 要访问的图片文件通过系统调用 `mmap()` 映射到内存，通过 `CGImageSourceRef` 访问图像数据，创建`CGImageRef`。

- 传统操作系统的I/O操作为标准I/O，即缓存I/O。在这种I/O模型下，数据先从磁盘拷贝到内核空间的缓冲区，然后从内核空间缓冲区拷贝到用户的内存空间。这种方式的优点是减少了磁盘操作，提高性能。但因为数据在传输过程中需要在用户内存空间和内核空间间进行多次数据拷贝操作，造成很大的CPU及内存开销。
- `mmap()` 将硬盘数据直接映射到虚拟内存中，应用可以直接访问虚拟内存中对应的地址来读取数据，避免了数据在内核空间和用户空间的相互拷贝，效率更高。在**使用这些数据时，虚拟内存管理系统才会根据缺页加载的机制从磁盘加载对应的数据块到物理内存，在这之前不会消耗用户空间的内存。** iOS中，使用 imageNamed 或者imageWithContentsOfFile 时，系统会调用 mmap() 将图片文件映射到虚拟内存，并创建 CGImageRef 用于后续访问图片数据。



2. 在主线程中，将图片数据赋值给 UIImageView 。在保存图片时，为了节省空间，通常会将图片编码（压缩）后再进行存储。**如果读取的图片数据为压缩后的数据的话，那就需要对其进行解码成位图（Bitmap）数据。** 不同加载图片的方式，在这一步的操作上会有一定的差异。

- `imageNamed:` 会在图片第一次渲染到屏幕上的时候进行解码，并缓存解码后的图片数据。缓存数据存储在全局缓存中，不会随着UIImag的释放而释放。
- `imageWithContentsOfFile:` 或 `imageWithData:` 同样会在图片第一次渲染到屏幕上的时候进行解码。底层会调用到 `CGImageSourceCreateWithData()` 方法，该方法可以指定是否要缓存解码后的数据，在64位机器上默认需要缓存（`kCGImageSourceShouldCache`）。与上面的方法不同，这种方式创建的缓存会随着UIImage的释放而被释放掉。

3. 手动调用 `CGImageSourceCreateWithData()` 方法可以指定是否需要缓存（`kCGImageSourceShouldCache`），之后再调用 `CGImageSourceCreateImageAtIndex()` 可以设置是否需要立即进行解码（`kCGImageSourceShouldCacheImmediately`），如果设置为不需要立刻解码，则会在**将图片渲染到屏幕上时才进行解码。**（设置为立即解码会阻塞主线程，造成性能问题，详见 https://www.objc.io/issues/5-ios7/iOS7-hidden-gems-and-workarounds/）
4. UIImageView 的图层树（Layer Tree）发生变化，会生成一个 `Implicit Transaction`，这个`transaction`会自动在主线程的下一个 Runloop 进行提交。（`Explicit Transaction` 由显式调用 begin() 和 commit() 方法触发生成。）
5. 下一个Main Runloop中，Core Animation会提交这个 Implicit Transaction。如果用户内存中的位图数据没**有字节对齐** ，出于渲染性能考虑， **Core Animation会对数据进行拷贝，以进行字节对齐。** 之后，GPU会渲染对齐后的位图数据，展示在屏幕上。

