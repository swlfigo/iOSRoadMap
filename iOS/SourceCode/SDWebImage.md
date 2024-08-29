# SDWebImage



一个为UIImageView提供一个分类来支持远程服务器图片加载的库。

## 功能简介：

```
      1、一个添加了web图片加载和缓存管理的UIImageView分类
      2、一个异步图片下载器
      3、一个异步的内存加磁盘综合存储图片并且自动处理过期图片
      4、支持动态gif图
      5、支持webP格式的图片
      6、后台图片解压处理
      7、确保同样的图片url不会下载多次
      8、确保伪造的图片url不会重复尝试下载
      9、确保主线程不会阻塞
```



## View Category:



所有控件设置图片的方法，最终都会来到 UIView+WebCache 分类下：

```
- (void)sd_internalSetImageWithURL:(nullable NSURL *)url
                  placeholderImage:(nullable UIImage *)placeholder
                           options:(SDWebImageOptions)options
                           context:(nullable SDWebImageContext *)context
                     setImageBlock:(nullable SDSetImageBlock)setImageBlock
                          progress:(nullable SDImageLoaderProgressBlock)progressBlock
                         completed:(nullable SDInternalCompletionBlock)completedBlock;
```



1. 利用 copy 将 SDWebImageContext 复制并转换为不可变类型。 validOperationKey 值作为校验 id，默认值为当前 view 的类名。
2. **sd_cancelImageLoadOperationWithKey**：取消上一次任务，保证没有当前正在进行的异步下载操作, 不会与即将进行的操作发生冲突。**保证当前的控件**上有且只有一个最新的任务。
   - 根据传入的 context(字典) 找到当前 validOperationKey，一般 context 为 nil，会自动创建。然后会将当前实例的类名作为 validOperationKey。
   - 在 UIView+WebCacheOperation 分类中，设置了一个关联属性 SDOperationsDictionary。它会存储当前实例的所有 operation 操作。
   - 在实例开始真正的图片请求操作之前，会根据 validOperationKey 获取 operation 操作，如果之前有操作存在，则会取消之前的操作，保证当前实例执行的是最新的 operation。
3. 设置占位图。
4. 重置 NSProgress、 设置 SDWebImageIndicator，并判断是否开启。
5. 初始化 SDWebImageManager 、SDImageLoaderProgressBlock。
6. 利用 SDWebImageManager 开启下载 loadImageWithURL: 并将返回的 SDWebImageOperation 存入 sd_operationDictionary，key 为 validOperationKey。
7. 取到图片后，停止 indicator。调用 sd_setImage: 同时为新的 image 添加 Transition 过渡动画。



**说明**

SDOperationsDictionary 是一个 **strong——weak** 的 NSMapTable，对 operation 拥有一个弱引用，方便 cancel。其强引用由 SDWebImageManager 的 runningOperations 保持。

```
typedef NSMapTable<NSString *, id<SDWebImageOperation>> SDOperationsDictionary;

[[NSMapTable alloc] initWithKeyOptions:NSPointerFunctionsStrongMemory 
						  valueOptions:NSPointerFunctionsWeakMemory 
                          	  capacity:0];
```

使用weak，在后续operation下载操作回调后,获取这个operation。如果View已经重用或者消失，则不会设置图片避免混乱。





## SDImageManager

### 属性介绍

SDImageManager 是整个框架的中心，所有的处理逻辑都在这里面进行组装、分发。

```objective-c
@property (nonatomic, class, readonly, nonnull) SDWebImageManager *sharedManager;

@property (weak, nonatomic, nullable) id <SDWebImageManagerDelegate> delegate;

@property (strong, nonatomic, readonly, nonnull) id<SDImageCache> imageCache;	//缓存处理

@property (strong, nonatomic, readonly, nonnull) id<SDImageLoader> imageLoader;	//图片下载器

@property (strong, nonatomic, nullable) id<SDImageTransformer> transformer;	//用于在图像加载完成后进行图像变换，并将变换后的图像存储到缓存中。

@property (nonatomic, strong, nullable) id<SDWebImageCacheKeyFilter> cacheKeyFilter;	//默认情况下，是把 URL.absoluteString 作为 cacheKey ，而如果设置了 fileter 则会对通过 cacheKeyForURL: 对 cacheKey 拦截并进行修改。

@property (nonatomic, strong, nullable) id<SDWebImageCacheSerializer> cacheSerializer; //默认情况下，ImageCache 会直接将 downloadData 进行缓存，而当我们使用其他图片格式进行传输时，例如 WEBP 格式的，那么磁盘中的存储则会按 WEBP 格式来。这会产生一个问题，每次当我们需要从磁盘读取 image 时都需要进行重复的解码操作。而通过 CacheSerializer 可以直接将 downloadData 转换为 JPEG/PNG 的格式的 NSData 缓存，从而提高访问效率。

@property (nonatomic, strong, nullable) id<SDWebImageOptionsProcessor> optionsProcessor;	//用于全局控制当前管理器的 SDWebImageOptions 和 SDWebImageContext 中的参数。

@property (nonatomic, assign, readonly, getter=isRunning) BOOL running;	//标识当前 manager 是否有 operation 正在运行。内部维护了 runningOperations 集合，当数量大于 0 时，说明有操作在执行。

@property (nonatomic, class, nullable) id<SDImageCache> defaultImageCache; //默认使用 SDImageCache.sharedImageCache。

@property (nonatomic, class, nullable) id<SDImageLoader> defaultImageLoader; //默认使用 SDWebImageDownloader.sharedDownloader。


//Delegate
/**
 判断当前 url 是否需要下载。默认为 true。
*/
- (BOOL)imageManager:(nonnull SDWebImageManager *)imageManager shouldDownloadImageForURL:(nonnull NSURL *)imageURL;

/**
  当下载失败之后，如果实现了这个代理，则将失败的 url 处理逻辑交给代理处理。
 */
- (BOOL)imageManager:(nonnull SDWebImageManager *)imageManager shouldBlockFailedURL:(nonnull NSURL *)imageURL withError:(nonnull NSError *)error;

```



### 主要方法

#### 入口

通过上层 Category 的封装之后，最终图片的加载逻辑会来到 SDWebImageManager 的这个方法：

```objective-c
- (nullable SDWebImageCombinedOperation *)loadImageWithURL:(nullable NSURL *)url
                                                   options:(SDWebImageOptions)options
                                                   context:(nullable SDWebImageContext *)context
                                                  progress:(nullable SDImageLoaderProgressBlock)progressBlock
                                                 completed:(nonnull SDInternalCompletionBlock)completedBlock;
```



```objective-c
@property (strong, nonatomic, nonnull) NSMutableSet<NSURL *> *failedURLs;
@property (strong, nonatomic, nonnull) dispatch_semaphore_t failedURLsLock; // a lock to keep the access to `failedURLs` thread-safe
@property (strong, nonatomic, nonnull) NSMutableSet<SDWebImageCombinedOperation *> *runningOperations;
@property (strong, nonatomic, nonnull) dispatch_semaphore_t runningOperationsLock; // a lock to keep the access to `runningOperations` thread-safe
```

这四个是在 SDWebImageManager 的 .m 文件中的 Extension 中声明的。

1. **failedURLs：** 保存了失败的请求 url。
2. **runningOperations：**会将在上面的方法中会生成的一个 SDWebImageCombinedOperation 实例，保存在集合中。图片加载存在两种情况，一种是直接在缓存中获取，一种是通过网络在下载，都会返回一个 NSOperation 对象，所以 SDWebImageCombinedOperation 实例中有两个属性与之一一对应，方便对两种加载图片的方式进行管理。
3. 利用信号量 dispatch_semaphore_t 防止多线程竞争。



**方法的执行的流程：**

1. url 合法性判断。因为，这里的 url 是 nullable 的。如果是 NSString 还会将其转换为 NSURL。
2. 生成 SDWebImageCombinedOperation 实例对象。
3. failedURLs 集合查询。
   - 若命中，且 options 不为 SDWebImageRetryFailed，则直接返回 operation 并 return。
   - 若未命中，或者 options 为 SDWebImageRetryFailed。则将 operation 存入 runningOperations。
4. 将 options 和 imageContext 封装为 SDWebImageOptionsResult。
5. 开始缓存查询。



### 缓存查询

```objective-c
- (void)callCacheProcessForOperation:(nonnull SDWebImageCombinedOperation *)operation
                                 url:(nonnull NSURL *)url
                             options:(SDWebImageOptions)options
                             context:(nullable SDWebImageContext *)context
                            progress:(nullable SDImageLoaderProgressBlock)progressBlock
                           completed:(nullable SDInternalCompletionBlock)completedBlock;
```

**方法的执行的流程：**

1. 确定用于查找缓存的实例对象。默认的 [SDImageCache sharedImageCache] 还是由 context 传入 SDWebImageContextImageCache。
2. 根据 options 参数确定是否需要查找缓存。 SDWebImageFromLoaderOnly
3. 根据 context 参数 SDWebImageContextQueryCacheType 确定缓存查找的范围。默认为 SDImageCacheTypeAll。
4. 需要查找缓存。
   - 根据 url 确定最终查找时使用的 key 值。可能由 cacheKeyFilter 进行变换。开始查找缓存。
   - 缓存查询结束后。
     - 判断 operation 是否被 cancel。如果是返回错误并结束。
     - operation 正常，进入下载。
5. 不需要查找缓存，直接进入下载。

#### 内存缓存 SDMemoryCache

1. 继承自 NSCache 实现内存缓存。通过双向链表及字典实现 LRU 的缓存策略。内存清理策略：对象数量 count、对象大小 cost 。
2. 维护了一个 NSMapTable 类型的 weakCache（strong-weak）又存储了一份缓存。

外部传入一个需要缓存的对象时，其引用计数为 1，SDMemoryCache 对其进行缓存时，会强引用被缓存的对象，使它的引用计数变为 2。此时，若 SDMemoryCache 清理了缓存，被缓存对象的引用计数减一，但是它还在内存中，但是，从 SDMemoryCache 中已经取不到这个对象了。为了解决这个问题，SDMemoryCache 在继承自 NSCache 的基础上，维护了一个 NSMapTable 属性 weakCache（stong-weak cache），它会弱引用被缓存对象，当缓存被清理之后，我们还可以在 weakCache 中获取到被缓存对象，就算对象被释放，因为弱引用也不会造成野指针问题。这是典型的 “空间换时间” 的思想。当然，针对 weakCache 的读写安全，也使用了 weakCacheLock （dispatch_semaphore_t）线程锁。

#### 磁盘缓存

1. 当内存中未命中缓存，则在一个串行队列 ioQueue 中同步或者异步地执行磁盘查询。

```objective-c
// 串行队列
_ioQueue = dispatch_queue_create("com.hackemist.SDImageCache", DISPATCH_QUEUE_SERIAL);

// 判断是同步查询还是异步查询
BOOL shouldQueryDiskSync = ((image && options & SDImageCacheQueryMemoryDataSync) ||
                                (!image && options & SDImageCacheQueryDiskDataSync));
```

- 因为磁盘缓存读取时，会产生许多临时变量，为了避免内存过高，使用 @autoreleasepool 包裹磁盘读取的代码。
- 只有当从磁盘取到缓存时，才会对图片进行解码。
- 利用这个全局声明的变量 SDImageCacheDecodeImageData，进行了图片解码的处理。
  - 在磁盘中根据 filePath 取出 imageData。
  - 利用 CGImageSourceCreateWithData 将 imageData 转换为 image。
  - 利用 SDImageCoderHelper 将 image 强制解码并返回解码后的图片。
- 将解码后的图片缓存到内存缓存中，然后通过 block 回调到 SDWebImageManager。

```objective-c
UIImage * _Nullable SDImageCacheDecodeImageData(NSData * _Nonnull imageData, 
                                              NSString * _Nonnull cacheKey, 
                                                SDWebImageOptions options, 
                                    SDWebImageContext * _Nullable context);
```

### 



### 下载数据

```
- (void)callDownloadProcessForOperation:(nonnull SDWebImageCombinedOperation *)operation
                                    url:(nonnull NSURL *)url
                                options:(SDWebImageOptions)options
                                context:(SDWebImageContext *)context
                            cachedImage:(nullable UIImage *)cachedImage
                             cachedData:(nullable NSData *)cachedData
                              cacheType:(SDImageCacheType)cacheType
                               progress:(nullable SDImageLoaderProgressBlock)progressBlock
                              completed:(nullable SDInternalCompletionBlock)completedBlock;
```

**方法的执行的流程：**

1. 确定用于下载的实例对象。默认的 [SDWebImageDownloader sharedDownloader] 还是 由 context 传入 SDWebImageContextImageLoader。
2. 检查是否需要开启下载。

```
BOOL shouldDownload = !SD_OPTIONS_CONTAINS(options, SDWebImageFromCacheOnly);
shouldDownload &= (!cachedImage || options & SDWebImageRefreshCached);
shouldDownload &= (![self.delegate respondsToSelector:@selector(imageManager:shouldDownloadImageForURL:)] || [self.delegate imageManager:self shouldDownloadImageForURL:url]);
shouldDownload &= [imageLoader canRequestImageForURL:url];
```

- 检查 options 值是否为 SDWebImageFromCacheOnly 或 SDWebImageRefreshCached。
- 由代理决定是否需要新建下载任务。
- 通过 imageLoader 控制能否支持下载任务。

1. 如果 shouldDownload 为 NO，则结束下载并调用 callCompletionBlockForOperation 与 safelyRemoveOperationFromRunning。此时如果存在 cacheImage 则会随 completionBlock 一起返回。
2. 如果 shouldDownload 为 YES，新建下载任务并将其保存在 combineOperation 的 loaderOperation。在新建任务前，如有取到 cacheImage 且 SDWebImageRefreshCached 为 YES，会将其存入 imageContext (没有则创建 imageContext)。
   - SDWebImageDownloader 中，维护了一个 NSOperationQueue 实例 _downloadQueue，默认的最大并发数为 6。还维护了可变字典 _URLOperations，key 为下载 url，value 为下载的 NSOperation 实例。
     - _downloadQueue 中利用 NSOperationQueue 的 addDependency 方法，使原队列中 operations 依赖于最新加入的 operation。实现了一个 LILO (后进先出) 的操作队列。
   - 在 _URLOperations 中，根据下载 url 获取 operation。
     - 如果 (operation == nil || operation.isFinished || operation.isCancelled) 则会创建一个新的 operation。 利用 @synchronized 为 operation 添加 block 回调（progressBlock， completedBlock），然后，将 operation 加入到 _URLOperations 字典中。
     - 否则，重用之前的 operation，利用 @synchronized 为 operation 添加 block 回调（progressBlock， completedBlock），并设置当前 operation 的操作优先级。
   - 根据获取到的 operation 生成 SDWebImageDownloadToken 实例并返回。在 SDWebImageDownloaderOperation 的完成回调中，可以看到也使用了 SDImageLoaderDecodeImageData 对图片进行了子线程强制解码并将解码后的 image 返回。

```
UIImage * _Nullable SDImageLoaderDecodeImageData(NSData * _Nonnull imageData, 
                                                  NSURL * _Nonnull imageURL, 
                                                 SDWebImageOptions options, 
                                     SDWebImageContext * _Nullable context);
```

1. 下载结束后回到 callBack，这里会先处理几种情况：
   - operation 被 cancel 则抛弃下载的 image、data ，callCompletionBlock 结束下载。
   - reqeust 被 cancel 导致的 error，callCompletionBlock 结束下载。
   - imageRefresh 后请求结果仍旧命中了 NSURLCache 缓存，则不会调用 callCompletionBlock。
   - error 出错，callCompletionBlockForOperation 并将 url 添加至 failedURLs。
   - 均无以上情况，如果是通过 retry 成功的，会先将 url 从 failedURLs 中移除，调用 storeCacheProcess。
   - 最后会对标记为 finished。执行 safelyRemoveOperation。



### 缓存数据

```
- (void)callStoreCacheProcessForOperation:(nonnull SDWebImageCombinedOperation *)operation
                                      url:(nonnull NSURL *)url
                                  options:(SDWebImageOptions)options
                                  context:(SDWebImageContext *)context
                          downloadedImage:(nullable UIImage *)downloadedImage
                           downloadedData:(nullable NSData *)downloadedData
                                 finished:(BOOL)finished
                                 progress:(nullable SDImageLoaderProgressBlock)progressBlock
                                completed:(nullable SDInternalCompletionBlock)completedBlock;
```

**方法的执行的流程：**

1. 先从 imageContext 中取出 storeCacheType、originalStoreCacheType、transformer、cacheSerializer，判断是否需要存储转换后图像数据、原始数据、等待缓存存储结束。
2. 检查是否需要缓存原始数据 shouldCacheOriginal。
   - shouldCacheOriginal = YES：先确认存储类型是否为原始数据，存储时如果 cacheSerializer 存在则会先转换数据格式，最终都调用 [self stroageImage:] 将数据存入缓存，并进入 image transformer。
   - shouldCacheOriginal = NO：直接进入 image transformer。



## SDWebImage常见问题



a. 如何避免同一时间多个请求，请求同一张图片下载多次问题。
b. 如何解决TableViewCell 复用时导致的图片展示错乱问题。



当我们使用SDWebImage加载图片时需要调用如下方法：

```
- (void)sd_setImageWithURL:(nullable NSURL *)url {
    [self sd_setImageWithURL:url placeholderImage:nil options:0 progress:nil completed:nil];
}
```



之后进行一系列的传递会传递到最深层的方法：

```
- (void)sd_setImageWithURL:(nullable NSURL *)url placeholderImage:(nullable UIImage *)placeholder
                                                options:(SDWebImageOptions)options
                                                context:(nullable SDWebImageContext *)context
                                                progress:(nullable SDImageLoaderProgressBlock)progressBlock
                                                completed:(nullable SDExternalCompletionBlock)completedBlock {
    [self sd_internalSetImageWithURL:url placeholderImage:placeholder options:options context:context setImageBlock:nil 
                                progress:progressBlock 
                                completed:^(UIImage * _Nullable image, NSData * _Nullable data, NSError * _Nullable error, 
                                SDImageCacheType cacheType, BOOL finished, NSURL * _Nullable imageURL) {
                    if (completedBlock) {
                        completedBlock(image, error, cacheType, imageURL);
                    }
                }];
}
```



可以看到，这个方法里面调用了UIView+Webcache分类里面的一个方法：

```
- (void)sd_internalSetImageWithURL:(nullable NSURL *)url placeholderImage:(nullable UIImage *)placeholder
                                                        options:(SDWebImageOptions)options
                                                        context:(nullable SDWebImageContext *)context
                                                        setImageBlock:(nullable SDSetImageBlock)setImageBlock
                                                        progress:(nullable SDImageLoaderProgressBlock)progressBlock
                                                        completed:(nullable SDInternalCompletionBlock)completedBlock {
                                                        ......
                                                        }
```



这个方法就是我们加载图片的正式入口方法。下面我们看一下这个方法里面都主要做了什么。
第一步，根据validOperationKey 取消掉正在执行的操作operation如下调用：

```
NSString *validOperationKey = context[SDWebImageContextSetImageOperationKey];
if (!validOperationKey) {
    validOperationKey = NSStringFromClass([self class]);
}
self.sd_latestOperationKey = validOperationKey;
[self sd_cancelImageLoadOperationWithKey:validOperationKey];
```



sd_cancelImageLoadOperationWithKey: 方法的内部实现会查询到已经存在的同名任务，并且会取消掉这个任务，并在当前view的operationDictionary 容器中移除掉。源码如下：

```
- (void)sd_cancelImageLoadOperationWithKey:(nullable NSString *)key {
if (key) {
    // Cancel in progress downloader from queue
    SDOperationsDictionary *operationDictionary = [self sd_operationDictionary];
    id<SDWebImageOperation> operation;

    @synchronized (self) {
        operation = [operationDictionary objectForKey:key];
    }
    if (operation) {
    if ([operation conformsToProtocol:@protocol(SDWebImageOperation)]) {
        [operation cancel];
    }
    @synchronized (self) {
        [operationDictionary removeObjectForKey:key];
    }
    }
  }
}
```



这里需要说明一下：[self sd_operationDictionary]这个调用，这个方法的实现是给当前View通过关联对象的技术关联了一个NSMapTable对象，用来存储请求链接接对应的请求操作类型如NSMapTable<NSString *, id>。源码如下：

```
- (SDOperationsDictionary *)sd_operationDictionary {
    @synchronized(self) {
        SDOperationsDictionary *operations = objc_getAssociatedObject(self, &loadOperationKey);
        if (operations) {
            return operations;
        }
        operations = [[NSMapTable alloc] initWithKeyOptions:NSPointerFunctionsStrongMemory valueOptions:NSPointerFunctionsWeakMemory capacity:0];
        objc_setAssociatedObject(self, &loadOperationKey, operations, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
        return operations;
    }
}
```



绕了这么大一圈，你可能会问，为什么一上来要调用sd_cancelImageLoadOperationWithKey:这个方法？通过上面的源码分析SDWebImage这样设计是**为了解决TableViewCell复用**时，如果被复用的Cell的ImageView请求的图片没有回调时展示图片错乱的问题。原理就是**如果被复用的Cell的ImageView之前请求的图片还没有回调，而此时需要请求新的图片，那么就取消掉之前的请求operation,并从operationDictionary中移除掉。然后去加载需要加载的新图片。如果说，之前的图片请求在这之后回调回来的话，会判断之前请求的operation是否存在，以及operation的isCancel属性，如果不存在或者isCancel=Yes的话，就不会回调到UI界面**。也就是如下代码逻辑：

```
@weakify(operation);
operation.loaderOperation = [self.imageLoader requestImageWithURL:url options:options context:context progress:progressBlock completed:^(UIImage *downloadedImage, NSData *downloadedData, NSError *error, BOOL finished) {
    @strongify(operation);
    if (!operation || operation.isCancelled) {
        // Do nothing if the operation was cancelled
        // See #699 for more details
        // if we would call the completedBlock, there could be a race condition between this block and another completedBlock for the same object, so if this one is called second, we will overwrite the new data
    }
```



说了这么多，相信应该清楚为什么要调用sd_cancelImageLoadOperationWithKey:方法了，我们接着回到sd_internalSetImageWithURL:方法中，cancel之后就会清掉当前imageView上次下载的图片：

```
if (!(options & SDWebImageDelayPlaceholder)) {
    dispatch_main_async_safe(^{
        [self sd_setImage:placeholder imageData:nil basedOnClassOrViaCustomSetImageBlock:setImageBlock cacheType:SDImageCacheTypeNone imageURL:url];
    });
}
```



这里可以解释，复用的时候，已经展示过图片的imageView为什么在被复用的时候没有展示之前存在的图片而是展示placeholer或者不展示的原因。
接下来，就是判断我们传入的url是否合法，以及设置UIImageView的加载指示器，还有加载进度block，此处不做详细说明了。我们着重看加载图片的方法：

```
id <SDWebImageOperation> operation = [manager loadImageWithURL:url options:options context:context progress:combinedProgressBlock completed:^(UIImage *image, NSData *data, NSError *error, SDImageCacheType cacheType, BOOL finished, NSURL *imageURL) {
    ......
}
```



这里当前view利用前面生成的manager 去加载我们需要的图片，并把获取的结果回调给了上一级调用方。从上面的代码可以看到，获取图片的同时返回了一个operation，这个operation就是标识获取当前url图片的一个操作。之后会把这个operation放在当前view的operationDictionary中：

```
[self sd_setImageLoadOperation:operation forKey:validOperationKey];
```



sd_setImageLoadOperation：内部实现如下:

```
- (void)sd_setImageLoadOperation:(nullable id<SDWebImageOperation>)operation forKey:(nullable NSString *)key {
    if (key) {
        [self sd_cancelImageLoadOperationWithKey:key];
        if (operation) {
            SDOperationsDictionary *operationDictionary = [self sd_operationDictionary];
            @synchronized (self) {
                [operationDictionary setObject:operation forKey:key];
            }
        }
    }
}
```



这也是程序一开始时，能够取消掉同名operation的原因。就是同一个view发送一个图片请求就会记录在operationDictionary中来标识有请求正在执行。
我们接着看loadImageWithURL:方法内部实现：
首先，判断url是否合法，然后生成一个请求图片的operation，这个和我们刚才讲到的operation在内存中是同一个，因为是从该方法中返回出去的。
其次，将这个operation添加到正在运行的操作容器中：

```
SD_LOCK(self.runningOperationsLock);
[self.runningOperations addObject:operation];
SD_UNLOCK(self.runningOperationsLock);
```



之后进入重点，那就是开始从缓存中读取图片：

```
// Start the entry to load image from cache
[self callCacheProcessForOperation:operation url:url options:options context:context progress:progressBlock completed:completedBlock];
```



同样的，将我们刚才讲到的operation传入到这个方法中。我们看一下这个方法中做了什么：

```
// Query cache process
- (void)callCacheProcessForOperation:(nonnull SDWebImageCombinedOperation *)operation 
                                url:(nonnull NSURL *)url 
                                options:(SDWebImageOptions)options
                                context:(nullable SDWebImageContext *)context
                                progress:(nullable SDImageLoaderProgressBlock)progressBlock
                                completed:(nullable SDInternalCompletionBlock)completedBlock {
        // Check whether we should query cache
        BOOL shouldQueryCache = (options & SDWebImageFromLoaderOnly) == 0;
        if (shouldQueryCache) {
            id<SDWebImageCacheKeyFilter> cacheKeyFilter = context[SDWebImageContextCacheKeyFilter];
            NSString *key = [self cacheKeyForURL:url cacheKeyFilter:cacheKeyFilter];
            @weakify(operation);
            operation.cacheOperation = [self.imageCache queryImageForKey:key options:options context:context completion:^(UIImage * _Nullable cachedImage, NSData * _Nullable cachedData, SDImageCacheType cacheType) {
                @strongify(operation);
                if (!operation || operation.isCancelled) {
                [self safelyRemoveOperationFromRunning:operation];
                return;
            }
            // Continue download process
            [self callDownloadProcessForOperation:operation url:url options:options context:context cachedImage:cachedImage cachedData:cachedData cacheType:cacheType progress:progressBlock completed:completedBlock];
        }];
        } else {
        // Continue download process
        [self callDownloadProcessForOperation:operation url:url options:options context:context cachedImage:nil cachedData:nil cacheType:SDImageCacheTypeNone progress:progressBlock completed:completedBlock];
    }
}
```



从上面的源码可以看出，
首先判断是否需要从缓存中读取图片，如果需要，就处理url,处理后得到我们读取缓存的key。
然后，开始从缓存中读取图片，回调之后判断当前operation是否还存在，以及operation是否被取消，如果取消的话就从runningOperations中移除当前operation并返回，什么也不做。否则，调用下载处理程序：callDownloadProcessForOperation：并把我们读取出来的缓存数据传入该方法。接下来我们看看这个方法的内部实现：
首先判断是否需要下载图片，如果不需要就判断缓存数据如果缓存有值就直接返回给调用方，如果需要就先看一下之前读取的缓存数据是否有值，如果有值，就直接返回给调用方。如果没有的话，就使用imageLoader下载图片：

```
// `SDWebImageCombinedOperation` -> `SDWebImageDownloadToken` -> `downloadOperationCancelToken`, which is a `SDCallbacksDictionary` and retain the completed block below, so we need weak-strong again to avoid retain cycle
@weakify(operation);
operation.loaderOperation = [self.imageLoader requestImageWithURL:url options:options context:context progress:progressBlock completed:^(UIImage *downloadedImage, NSData *downloadedData, NSError *error, BOOL finished) {
    @strongify(operation);
    if (!operation || operation.isCancelled) {
        // Do nothing if the operation was cancelled
        // See #699 for more details
        // if we would call the completedBlock, there could be a race condition between this block and another completedBlock for the same object, so if this one is called second, we will overwrite the new data
    } else if (cachedImage && options & SDWebImageRefreshCached && [error.domain isEqualToString:SDWebImageErrorDomain] && error.code == SDWebImageErrorCacheNotModified) {
        // Image refresh hit the NSURLCache cache, do not call the completion block
    } else if (error) {
        [self callCompletionBlockForOperation:operation completion:completedBlock error:error url:url];
        BOOL shouldBlockFailedURL = [self shouldBlockFailedURLWithURL:url error:error];

        if (shouldBlockFailedURL) {
            SD_LOCK(self.failedURLsLock);
            [self.failedURLs addObject:url];
            SD_UNLOCK(self.failedURLsLock);
        }
    } else {
        if ((options & SDWebImageRetryFailed)) {
            SD_LOCK(self.failedURLsLock);
            [self.failedURLs removeObject:url];
            SD_UNLOCK(self.failedURLsLock);
        }
        [self callStoreCacheProcessForOperation:operation url:url 
                        options:options context:context 
                        downloadedImage:downloadedImage 
                        downloadedData:downloadedData 
                        finished:finished 
                        progress:progressBlock 
                        completed:completedBlock];
    }   

    if (finished) {
        [self safelyRemoveOperationFromRunning:operation];
    }
}];
```



从上面的源码中可以看出请求图片的回调回来后：
1.如果operation不存在或者被取消，什么也不处理
2.如果有error则直接回调错误信息，并把当前url加入到filedURLs中。
3.如果一切正常，则把错误请求从filedURLs中移除，并把下载好的图片数据传递到缓存处理程序。
4.最后，如果finished==YES，则把当前operation从runningOperations中移除。

接下来我们看一下这个方法的内部实现：
首先处理一些下载器选项，然后调用下载图片方法：

```
return [self downloadImageWithURL:url options:downloaderOptions context:context progress:progressBlock completed:completedBlock];
```



接着看上面这个方法的内部实现：
首先判断url是否合法，如果合法，从下载器的URLOperations属性中读取该url对应的operation，如果operation不存在，或者已经取消或者已经完成，则根据url重新生成一个operation,同时记录该operation到URLOperations中，并把该operation添加到下载队列中去：

```
self.URLOperations[url] = operation;
// Add operation to operation queue only after all configuration done according to Apple's doc.
// `addOperation:` does not synchronously execute the `operation.completionBlock` so this will not cause deadlock.
[self.downloadQueue addOperation:operation];
```



如果存在operation，但是operation没有正在执行，则根据条件调整operation的请求优先级。
如果有正在执行的operation，不创建新的请求operation，而是给当前operation添加回调对象progressBlock 和 completedBlock。

```
id downloadOperationCancelToken = [operation addHandlersForProgress:progressBlock completed:completedBlock];
```

看下这个方法的内部实现：

```
- (nullable id)addHandlersForProgress:(nullable SDWebImageDownloaderProgressBlock)progressBlock
completed:(nullable SDWebImageDownloaderCompletedBlock)completedBlock {
    SDCallbacksDictionary *callbacks = [NSMutableDictionary new];
    if (progressBlock) callbacks[kProgressCallbackKey] = [progressBlock copy];
    if (completedBlock) callbacks[kCompletedCallbackKey] = [completedBlock copy];
    SD_LOCK(self.callbacksLock);
    [self.callbackBlocks addObject:callbacks];
    SD_UNLOCK(self.callbacksLock);
    return callbacks;
}
```



从中可以看出一个ImageDownloaderOperation可以有多个回调block。
那么问题来了，SDWebImage为什么会这么设计呢？
答案是为了解决在同一时间，多个请求同时下载一张图片的时候，对该图片请求只下载一次。也就是请求只发送一次，而请求有结果的时候根据存储的多个返回block 依次返回给调用方。这方法是不是很机智。这一点也可从请求结果的代码中得到验证：

```
- (void)callCompletionBlocksWithImage:(nullable UIImage *)image
imageData:(nullable NSData *)imageData
            error:(nullable NSError *)error
                finished:(BOOL)finished {
                NSArray<id> *completionBlocks = [self callbacksForKey:kCompletedCallbackKey];
                dispatch_main_async_safe(^{
                    for (SDWebImageDownloaderCompletedBlock completedBlock in completionBlocks) {
                        completedBlock(image, imageData, error, finished);
                    }
    });
}
```



从上面的代码中可以看到，方法内部是遍历了所有需要完成回调的completedBlock,然后回调出去。







## Reference

[SDWebImage源码学习 | 江涛的博客 (coderjtao.github.io)](https://coderjtao.github.io/2020/04/26/SDWebImage源码学习)

[SDWebImage (5.0.6) 图片加载奇淫巧技 | Charles' Blog (icloudart.com)](http://icloudart.com/2019/07/29/SDWebImage (5.0.6) 图片加载奇淫巧技/)

[SDWebImage (5.0.6) 图片缓存读写原理 | Charles' Blog (icloudart.com)](http://icloudart.com/2019/08/04/SDWebImage (5.0.6) 图片缓存读写原理/)