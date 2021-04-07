# KTVHttpCache

目前iOS端比较常见的视频缓存的实现方式主要有两种：
 1、使用iOS自带的AVURLAsset的AVAssetResourceLoader来实现。
 2、在客户端搭建local服务器，local服务器作为中间者，代替客户端请求服务器数据，并将获取到的数据缓存，再提供给客户端。
 我们项目里使用的是KTVHTTPCache来实现视频缓存，KTVHTTPCache的实现方式就是第二种，项目地址：([https://github.com/ChangbaDevs/KTVHTTPCache](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2FChangbaDevs%2FKTVHTTPCache))。

## 具体实现：

KTVHTTPCache的使用比较简单：



```objective-c
NSURL *proxyURL = [KTVHTTPCache proxyURLWithOriginalURL:originalURL];
AVPlayer *player = [AVPlayer playerWithURL:proxyURL];
```



可以看出，它是将源视频的URL替换成了自己定义格式的URL，这时我们其实请求的就是local服务器了。
 核心的流程大概是这样：

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-04-02-070358.jpg)



![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-04-07-140250.jpg)

KTVHTTPCache 由 HTTP Server 和 Data Storage 两大模块组成。前者负责与 Client 交互，后者负责资源加载及缓存处理。



几个核心类实现：

**1、KTVHCHTTPServer：**
 用来搭建local server的，内部使用第三方库HTTPServer实现:
 创建自己的Connection类继承自HTTPConnection

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-04-07-140701.jpg)

```objective-c
@interface KTVHCHTTPConnection : HTTPConnection
```



重写子类方法，返回相应的response类



```objective-c
- (NSObject<HTTPResponse> *)httpResponseForMethod:(NSString *)method URI:(NSString *)path
{
    KTVHCLogHTTPConnection(@"%p, Receive request\nmethod : %@\npath : %@\nURL : %@", self, method, path, request.url);
    NSDictionary<NSString *,NSString *> *parameters = [[KTVHCURLTool tool] parseQuery:request.url.query];
    NSURL *URL = [NSURL URLWithString:[parameters objectForKey:@"url"]];
    KTVHCDataRequest *dataRequest = [[KTVHCDataRequest alloc] initWithURL:URL headers:request.allHeaderFields];
    KTVHCHTTPResponse *response = [[KTVHCHTTPResponse alloc] initWithConnection:self dataRequest:dataRequest];
    return response;
}
```

创建response作为Local Server数据返回体，遵循HTTPResponse协议，实现协议方法

```objective-c
@interface KTVHCHTTPResponse : NSObject <HTTPResponse>
```



实现协议方法



```objective-c
#pragma mark - HTTPResponse
- (NSData *)readDataOfLength:(NSUInteger)length
{
  "读取数据最开始的入口"
   NSData *data = [self.reader readDataOfLength:length];
   KTVHCLogHTTPResponse(@"%p, Read data : %lld", self, (long long)data.length);
   if (self.reader.isFinished) {
       KTVHCLogHTTPResponse(@"%p, Read data did finished", self);
       [self.reader close];
       [self.connection responseDidAbort:self];
   }
   return data;
}
………………（省略，节省篇幅）
```

这样，当本地发生请求时，就会获取KTVHCHTTPResponse内部方法返回的数据。

 **2、KTVHCDataReader和KTVHCDataSourceManager**

 从服务器返回类可以看到，数据的入口是从KTVHCDataReader的readDataOfLength获取的。



```objective-c
#pragma mark - KTVHCDataReader
- (NSData *)readDataOfLength:(NSUInteger)length
{
    [self lock];
    if (self.isClosed) {
        [self unlock];
        return nil;
    }
    if (self.isFinished) {
        [self unlock];
        return nil;
    }
    if (self.error) {
        [self unlock];
        return nil;
    }
    NSData *data = [self.sourceManager readDataOfLength:length];
    if (data.length > 0) {
        self->_readedLength += data.length;
        if (self.response.contentLength > 0) {
            self->_progress = (double)self.readedLength / (double)self.response.contentLength;
        }
    }
    KTVHCLogDataReader(@"%p, Read data : %lld", self, (long long)data.length);
    if (self.sourceManager.isFinished) {
        KTVHCLogDataReader(@"%p, Read data did finished", self);
        self->_finished = YES;
        [self close];
    }
    [self unlock];
    return data;
}
```

从这个方法里我们可以看到，读取数据又走到了KTVHCDataSourceManager中去。



```objective-c
#pragma mark - KTVHCDataReader
- (void)prepareSourceManager
{
   "两个数组保存两种数据来源"
    NSMutableArray<KTVHCDataFileSource *> *fileSources = [NSMutableArray array];
    NSMutableArray<KTVHCDataNetworkSource *> *networkSources = [NSMutableArray array];
    long long min = self.request.range.start;
    long long max = self.request.range.end;
    NSArray *unitItems = self.unit.unitItems;
    for (KTVHCDataUnitItem *item in unitItems) {
        long long itemMin = item.offset;
        long long itemMax = item.offset + item.length - 1;
        if (itemMax < min || itemMin > max) {
            continue;
        }
        if (min > itemMin) {
            itemMin = min;
        }
        if (max < itemMax) {
            itemMax = max;
        }
        min = itemMax + 1;
        KTVHCRange range = KTVHCMakeRange(item.offset, item.offset + item.length - 1);
        KTVHCRange readRange = KTVHCMakeRange(itemMin - item.offset, itemMax - item.offset);
        KTVHCDataFileSource *source = [[KTVHCDataFileSource alloc] initWithPath:item.absolutePath range:range readRange:readRange];
        [fileSources addObject:source];
    }
    [fileSources sortUsingComparator:^NSComparisonResult(KTVHCDataFileSource *obj1, KTVHCDataFileSource *obj2) {
        if (obj1.range.start < obj2.range.start) {
            return NSOrderedAscending;
        }
        return NSOrderedDescending;
    }];
    "对比本地已缓存的数据和视频数据量"
   "除了本地的如果还有未获取的数据，就需要网络请求获取了"
    long long offset = self.request.range.start;
    long long length = KTVHCRangeIsFull(self.request.range) ? KTVHCRangeGetLength(self.request.range) : (self.request.range.end - offset + 1);
    for (KTVHCDataFileSource *obj in fileSources) {
        long long delta = obj.range.start + obj.readRange.start - offset;
        if (delta > 0) {
            KTVHCRange range = KTVHCMakeRange(offset, offset + delta - 1);
            KTVHCDataRequest *request = [self.request newRequestWithRange:range];
            KTVHCDataNetworkSource *source = [[KTVHCDataNetworkSource alloc] initWithRequest:request];
            [networkSources addObject:source];
            offset += delta;
            length -= delta;
        }
        offset += KTVHCRangeGetLength(obj.readRange);
        length -= KTVHCRangeGetLength(obj.readRange);
    } 
 
    if (length > 0) {
        KTVHCRange range = KTVHCMakeRange(offset, self.request.range.end);
        KTVHCDataRequest *request = [self.request newRequestWithRange:range];
        KTVHCDataNetworkSource *source = [[KTVHCDataNetworkSource alloc] initWithRequest:request];
        [networkSources addObject:source];
    }
    NSMutableArray<id<KTVHCDataSource>> *sources = [NSMutableArray array];
    [sources addObjectsFromArray:fileSources];
    [sources addObjectsFromArray:networkSources];
    self.sourceManager = [[KTVHCDataSourceManager alloc] initWithSources:sources delegate:self delegateQueue:self.internalDelegateQueue];
    [self.sourceManager prepare];
}
```

看到KTVHCDataSourceManager的初始化过程， 可以看出其实正常获取数据的是KTVHCDataFileSource和KTVHCDataNetworkSource两个类。
 再看一下KTVHCDataSourceManager的readDataOfLength方法：



```objective-c
#pragma mark - KTVHCDataSourceManager
- (NSData *)readDataOfLength:(NSUInteger)length
{
    [self lock];
    if (self.isClosed) {
        [self unlock];
        return nil;
    }
    if (self.isFinished) {
        [self unlock];
        return nil;
    }
    if (self.error) {
        [self unlock];
        return nil;
    }
    "从Source里读取数据"
    NSData *data = [self.currentSource readDataOfLength:length];

    self->_readedLength += data.length;
    KTVHCLogDataSourceManager(@"%p, Read data : %lld", self, (long long)data.length);
    if (self.currentSource.isFinished) {
        "一个source读完，切换到下一个Source"
        self.currentSource = [self nextSource];
        if (self.currentSource) {
            KTVHCLogDataSourceManager(@"%p, Switch to next source, %@", self, self.currentSource);
            if ([self.currentSource isKindOfClass:[KTVHCDataFileSource class]]) {
                [self.currentSource prepare];
            }
        } else {
            KTVHCLogDataSourceManager(@"%p, Read data did finished", self);
            self->_finished = YES;
        }
    }
    [self unlock];
    return data;
}
```

**KTVHCDataNetworkSource和KTVHCDataFileSource
 从名字就可以看出：这两个类，一个是负责从直接从本地文件提供数据，一个是负责从网络读取之后提供数据
 KTVHCDataFileSource的readDataOfLength实现比较明显，就是单纯从文件里读取数据。
 看下KTVHCDataNetworkSource：



```objective-c
- (void)ktv_download:(KTVHCDownload *)download didReceiveResponse:(KTVHCDataResponse *)response
{
    [self lock];
    if (self.isClosed || self.error) {
        [self unlock];
        return;
    }
    self->_response = response;
    NSString *path = [KTVHCPathTool filePathWithURL:self.request.URL offset:self.request.range.start];
    self.unitItem = [[KTVHCDataUnitItem alloc] initWithPath:path offset:self.request.range.start];
    KTVHCDataUnit *unit = [[KTVHCDataUnitPool pool] unitWithURL:self.request.URL];
    [unit insertUnitItem:self.unitItem];
    KTVHCLogDataNetworkSource(@"%p, Receive response\nResponse : %@\nUnit : %@\nUnitItem : %@", self, response, unit, self.unitItem);
    [unit workingRelease];
    "创建了两个文件句柄，读和写。"
    self.writingHandle = [NSFileHandle fileHandleForWritingAtPath:self.unitItem.absolutePath];
    self.readingHandle = [NSFileHandle fileHandleForReadingAtPath:self.unitItem.absolutePath];
    [self callbackForPrepared];
    [self unlock];
}

- (void)ktv_download:(KTVHCDownload *)download didReceiveData:(NSData *)data
{
    [self lock];
    if (self.isClosed || self.error) {
        [self unlock];
        return;
    }
    @try {
        "接收到数据之后，写入文件。"
        [self.writingHandle writeData:data];
        self.downloadLength += data.length;
        [self.unitItem updateLength:self.downloadLength];
        KTVHCLogDataNetworkSource(@"%p, Receive data : %lld, %lld, %lld", self, (long long)data.length, self.downloadLength, self.unitItem.length);
       "有可用数据了，需要回调通知。"
        [self callbackForHasAvailableData];
    } @catch (NSException *exception) {
        NSError *error = [KTVHCError errorForException:exception];
        KTVHCLogDataNetworkSource(@"%p, write exception\nError : %@", self, error);
        [self callbackForFailed:error];
        if (!self.downloadCalledComplete) {
            KTVHCLogDataNetworkSource(@"%p, Cancel download task when write exception", self);
            [self.downlaodTask cancel];
            self.downlaodTask = nil;
        }
    }
    [self unlock];
}
```

可以看出，两个source的实现比较类似，只不过KTVHCDataNetworkSource多了一个从网络获取数据写入文件的步骤，其实最终提供数据还是通过文件读取的方式。
 一旦有可用数据，就通过delegate的方式一直回调，通知response类有可用数据。



```objective-c
#pragma mark -  KTVHCHTTPResponse
- (void)ktv_readerDidPrepare:(KTVHCDataReader *)reader
{
    KTVHCLogHTTPResponse(@"%p, Prepared", self);
    if (self.reader.isPrepared && self.waitingResponse == YES) {
        KTVHCLogHTTPResponse(@"%p, Call connection did prepared", self);
        [self.connection responseHasAvailableData:self];
    }
}
"这个回调获取有可用的数据的通知。"

- (void)ktv_readerHasAvailableData:(KTVHCDataReader *)reader
{
    KTVHCLogHTTPResponse(@"%p, Has available data", self);
    "这个方法就会触发response的readDataOfLength"
    [self.connection responseHasAvailableData:self];
}

- (void)ktv_reader:(KTVHCDataReader *)reader didFailWithError:(NSError *)error
{
    KTVHCLogHTTPResponse(@"%p, Failed\nError : %@", self, error);
    [self.reader close];
    [self.connection responseDidAbort:self];
}
```



## Reference

[1. iOS 视频缓存KTVHTTPCache原理和实现](https://www.jianshu.com/p/7daec9ce6390)

[2. 读懂「 唱吧KTVHTTPCache 」设计思想](https://www.jianshu.com/p/2314782a16c3)

