> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/giXmBAC0ft-kRB3BloawzA)

作者：shishu

审核：gj,zsb,gbn,zjz

**引言**
------

你是否也碰到了启动图不更新、未加载等异常问题，今天就给大家带来一个终极解决方案。

Demo 地址：https://github.com/iversonxh/DynamicLaunchImage

效果图：  

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-04-12-142341.gif)

**一、背景和问题**
-----------

iOS 启动图相信大家都非常熟悉，版本迭代中不免会遇到更换启动图的需求，本以为这是件很简单的事情，但实际操作时却遇到了各种毫无头绪的异常问题，如启动图不更新、启动图未成功渲染等。

苹果曾在 2019 年 WWDC 上宣布自 2020 年 4 月起，提交审核的应用都必须使用 storyboard 来配置启动图。而步入 2020 年以来，苹果也多次发布公告要求更换启动图配置方式：

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-04-12-142355.png)

具体可点击链接查看：https://developer.apple.com/news/?id=03262020b

在此背景下，百度 App 随即开展了相关的更换工作，具体的`LaunchScreen.storyboard`配置方式不再赘述，我们直接说配置后出现的问题：

*   启动图未渲染成功，表现为每次启动均为白屏，并且线上也有复现，这是我们遇到的主要问题（该问题我们在某些知名 App 上也有复现）；
  
*   启动图未能更新，启动后仍展示旧启动图，这个问题相信有不少同学遇到。
  

**二、问题分析定位**
------------

首先我们怀疑是配置方式有误、编译缓存等导致的问题，所以针对这些猜测我们做了以下测试：

*   不同系统、不同机型测试，均有复现，排除该问题只发生在特定机型或系统上；
  
*   清空编译缓存，仍旧复现，故排除编译缓存问题；
  
*   给`imageView添加`背景色，启动时正常显示`imageView`的背景色，但图片内容未显示，故排除了布局问题；
  
*   将图片从`Assets`中迁移至工程根目录下，出现空白启动图概率降低，但仍会偶现；
  
*   修改图片名，前几次正常，之后依旧偶现；
  
*   卸载应用重新安装，大概率恢复正常，仍复现；
  
*   将`LaunchScreen.storyboard`文件复制到新建的空工程中，仍复现，此时猜测为系统缓存问题；
  
*   ……
  

经过一系列的测试，我们排除了人为因素、编译问题等可能出现问题的点，最终认定是系统问题导致。

接着我们想到当启动图出现问题时，系统是否会有一些辅助信息输出呢？果然通过 Mac 控制台应用，虽然没有找到明显的异常信息输出，但是我们从中发现了关于启动图生成的关键信息（以下测试基于`iOS13`系统，不同系统上表现存在差异）。

1.  我们创建一个空工程，设备方向默认不更改，配置好启动图：  
  
    ![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-04-12-142403.png)
    
2.  在【`Edit Scheme】-【``Run】-`【`Launch】，将其`设置为【`Wait for the executable to be launched】`，接着运行工程，在控制台应用中搜索 `SpringBoard` 找到如下信息：  
    ![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-04-12-142410.png)  
    
3.  从日志中我们了解到，应用安装后，SpringBoard 异步发起截图请求，接着由 SplashBoard.framework 生成截图，最后写入磁盘。
  
    Demo 中共生成四张截图，分别为对应着浅色主题下竖屏启动图、浅色主题下横屏启动图、深色主题下竖屏启动图、深色主题下横屏启动图，竖 / 横屏截图是否生成由 info.plist 中所支持的设备方向决定。  
    如果在 info.plist 中未勾选任何方向，那么系统会输出 “无法生成启动图，因为当前应用不支持任何有效的方向”，此种情况下系统生成启动图时机为首次启动应用时，大家可以自行实验下。  
    
4.  相信大家也注意到上图红框中的写入路径（路径较长截图中未能完全显示），查看完整输出如下：
  
    ```shell
    [baidu.TestLaunchScreen] Snapshot data for <XBApplicationSnapshot: 0x1098e10e0; …BEC9AEF7C41A> written to file: /private/var/mobile/Containers/Data/Application/573E7FE9-8A15-4E84-A562-F8C4A62EAFBC/Library/SplashBoard/Snapshots/baidu.TestLaunchScreen - {DEFAULT GROUP}/1FFD332B-EBA0-40C9-8EEE-BEC9AEF7C41A@3x.ktx
    
    [baidu.TestLaunchScreen] Snapshot data for <XBApplicationSnapshot: 0x115a8e7e0; …AFBB52DBDDB3> written to file: /private/var/mobile/Containers/Data/Application/573E7FE9-8A15-4E84-A562-F8C4A62EAFBC/Library/SplashBoard/Snapshots/baidu.TestLaunchScreen - {DEFAULT GROUP}/96920D11-6312-4D69-BBDB-AFBB52DBDDB3@3x.ktx
    
    [baidu.TestLaunchScreen] Snapshot data for <XBApplicationSnapshot: 0x108f97b60; …33E7BC4CFF46> written to file: /private/var/mobile/Containers/Data/Application/02CCE9FD-5F65-43F4-9D72-A5E0BA0C047E/Library/SplashBoard/Snapshots/baidu.TestLaunchScreen - {DEFAULT GROUP}/98F7B5B1-5B3B-478B-93A8-ED3DE6492AD1@3x.ktx
    [baidu.TestLaunchScreen] Snapshot data for <XBApplicationSnapshot: 0x1159be260; …479CC9CC8BAD> written to file: /private/var/mobile/Containers/Data/Application/573E7FE9-8A15-4E84-A562-F8C4A62EAFBC/Library/SplashBoard/Snapshots/baidu.TestLaunchScreen - {DEFAULT GROUP}/D9D48845-8565-42CE-A834-479CC9CC8BAD@3x.ktx
    ```
    
5.  此时看到写入路径正是我们所熟知的沙盒目录，接着我们将应用沙盒目录导出，查看`Library目录结构如下`：
  
    ```
    ├── Caches
    ├── Preferences
    └── SplashBoard
        └── Snapshots
            └── baidu.TestLaunchScreen\ -\ {DEFAULT\ GROUP}
                ├── 1FFD332B-EBA0-40C9-8EEE-BEC9AEF7C41A@3x.ktx
                ├── 96920D11-6312-4D69-BBDB-AFBB52DBDDB3@3x.ktx
                ├── 98F7B5B1-5B3B-478B-93A8-ED3DE6492AD1@3x.ktx
                └── D9D48845-8565-42CE-A834-479CC9CC8BAD@3x.ktx
    ```
    
    ![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-04-12-142433.png)
    
    果然，按照控制台中所输出的路径，我们找到了系统生成的启动图文件，其格式为 KTX。
    
    缓存启动图的文件名具有规则，但其规则我们不得而知。
    
6.  接着我们点击应用图标启动应用，再次观察控制台应用中输出：
  
    ![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-04-12-142437.png)如图可知，点击应用图标后，`SpringBoard`找到了一个可用的启动图，无需预热`SplashBoard`，直接使用可用的启动图。
    
7.  由以上分析我们知道系统启动应用时会检查当前是否有可用的启动图，所以我们猜想如果当前没有可用的启动图，那么应该会迫使系统重新生成。为此我们清空了缓存启动图，再次冷启应用，果然验证了我们的猜想：
  
    ![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-04-12-142441.png)上图中大致流程为，检测到无可用缓存启动图，预热 SplashBoard，生成新的启动图，并缓存至沙盒目录，而我们在沙盒目录中也找到了新生成的启动图文件。
    

根据以上的分析结果，我们知道应用启动时加载启动图的大致流程：

*   查找沙盒目录中是否存在可用的缓存启动图，如果有则直接使用，否则执行下一步；
  
*   根据 LaunchScreen.storyboard 生成新的启动图，并将其缓存至沙盒目录 / Library/SplashBoard/Snapshots/<Bundle identifier> - {DEFAULT GROUP}/。
  

但系统是如何生成的，调用了什么样的 API，我们无法得知，并且其生成时机也早于我们应用代码可控制时机，也就意味着我们无法控制系统生成启动图的行为，换句话说就是即使我们的 storyboard 文件配置无误，但启动图出现异常可能是无法避免的，所以我们的想法是既然无法从根源上避免启动图异常问题，那么我们是否能够提供补救措施，让其自动恢复正常，下次冷启就显示我们期望的启动图，这样不至于一旦出现异常后后续冷启都异常，对于用户来说也可接受。

所以接下来我们做了一些尝试来验证是否能够修复我们所遇到的问题：

1.  清空启动图缓存目录，迫使系统重新生成启动图文件，但仍出现白屏问题，方案无效；
  
2.  是否可以我们自己生成启动图放至缓存目录，让系统认为存在可用的缓存启动图：  
    a. 清空缓存目录，直接放入随意命名的图片，验证无效，系统会在应用下次启动时或应用挂起时，根据应用支持的界面方向及设备当前的方向重新生成对应的启动图；  
    b. 替换缓存启动图文件，即保证该目录下所有子文件名不变，但文件内容全部替换，**验证方案有效**：  
    ![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-04-12-142445.png)  
    替换后冷启效果:  
    
    ![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-04-12-142451.png)
    
3.  接着我们又做了多次测试，得出了以下结论：
  
    > a. 替换的图片名需与对应的缓存图完全一致，包括文件扩展名，但实际其内容格式可以为 `PNG`或`JPEG`。  
    > b. 替换的图片大小需与当前屏幕大小一致（图片宽高等于屏幕宽高或高宽），如果不一致，系统会重新生成缓存启动图。
    

**经过深度调研及不断地分析测试，我们终于得出一个可行方案，那就是替换系统生成的缓存启动图。**

**三、解决方案**

最终我们决定直接摒弃系统缓存的启动图，完全替换为我们自己生成的启动图。

即用户安装应用后，系统会自动生成启动图并缓存至沙盒目录，接着用户启动应用时，我们通过代码将沙盒目录下缓存的启动图文件全部替换为我们通过代码生成的启动图。

### 3.1 生成启动图

对 LaunchScreen.storyboard 的初始视图控制器进行截图，参考以下代码：

```objective-c
NSString *launchScreenName = [[[NSBundle mainBundle] infoDictionary] objectForKey:@"UILaunchStoryboardName"];
UIStoryboard *storyboard = [UIStoryboard storyboardWithName:launchScreenName bundle:nil];
UIViewController *vc = storyboard.instantiateInitialViewController;
UIGraphicsBeginImageContextWithOptions([UIScreen mainScreen].bounds.size, NO, [UIScreen mainScreen].scale);
[vc.view.layer renderInContext:UIGraphicsGetCurrentContext()];
UIImage *image = UIGraphicsGetImageFromCurrentImageContext();
```

3.2 替换启动图

前面提到替换图片时，需保持缓存目录下文件名不变，所以这里最简单的办法就是遍历缓存目录下的文件名，接着以这些文件名直接写入替换的图片。

然而当我们按照以上方案初步开发完成，进行多系统验证时，遇到了一个棘手的问题，测试发现方案在`iOS10.0`及以上工作正常替换成功，但是在**`iOS9.x`及以下系统方案无效**。通过断点调试发现调用`NSFileManager`接口获取缓存目录下的文件名列表为空，再通过观察控制台应用中的输出，发现根本原因是无读取权限：

> Sandbox: TestLaunchScreen(403) deny(1) file-read-data /private/var/mobile/Containers/Data/Application/E7CB1946-1CB2-48FF-9193-88FCF7848323/Library/Caches/Snapshots/baidu.TestLaunchScreen

接着我们又测试往缓存目录写入文件，发现也无写入权限：

> Sandbox: TestLaunchScreen(630) deny(1) file-write-create /private/var/mobile/Containers/Data/Application/1C4B15FB-6AE4-444F-96FA-9FC3B84622CD/Library/Caches/Snapshots/baidu.TestLaunchScreen1/test.png

**顺带测试了下在 iOS9.x 上删除该缓存目录，发现同样无权限。**

这里也是经过不断调试，找到了如下 `API 变相地实现了操作缓存目录`，大家可以查看 Demo 体会其作用：

```objective-c
- (BOOL)moveItemAtPath:(NSString *)srcPath 
                toPath:(NSString *)dstPath 
                 error:(NSError **)error API_AVAILABLE(macos(10.5), ios(2.0), watchos(2.0), tvos(9.0));
```

### 3.3 横竖屏问题

### 大部分非游戏类应用，支持的界面方向都只有竖屏（Portrait），即应用页面不会跟随设备方向旋转，始终以竖屏方向显示。但实际开发时，由于某些特殊需求，我们可能会勾选上横屏（LandScape Left / LandScape Right），虽然我们可以通过代码控制页面不跟随设备方向旋转，但是这会导致系统为应用分别生成横屏和竖屏的启动图，从而导致一个问题：

> 若用户未开启系统旋转锁定，且横置手机启动应用，这会使得应用启动时显示横屏方向的启动图，而部分应用并未考虑适配横屏场景启动图，从而可能导致该场景下启动图拉伸或压缩等显示异常，比如在 LaunchScreen.storyboard 中仅添加一张背景图，给其设置约束铺满全屏，竖屏时正常显示，但横屏时就异常了。（ps：大家可以关闭系统旋转锁定，参考横屏冷启淘宝及微信的解决方案）  
> 有一种解决方案是 info.plist 中 Supported interface orientations 置空，但这解决不了启动图不更新或无法渲染问题。

百度 App 正如上面所描述，我们的产品页面在 iPhone 上不会跟随设备方向旋转，但 iPad 上是需要支持设备方向旋转，所以我们的处理是：

> 针对 iPhone 上，我们通过代码仅生成竖屏启动图，然后直接替换全部的缓存启动图，即启动时不管设备方向如何，展示的始终为竖屏启动图；
> 
> 而针对 iPad 上，我们通过代码同时生成竖屏及横屏启动图，接着分别使用这两张图进行替换，同时在替换时进行校验，只有当替换的启动图与缓存启动图宽高一致时才执行，即竖屏只替换竖屏、横屏只替换横屏。

注意 iPad 上的方案涉及到图片宽高获取，而相信大家阅读到这里也知道了缓存图格式有`KTX`，但该图片无法直接使用`UIImage`接口进行加载，这里我们通过多机型、多系统地查看了`KTX`图片的元数据，发现总结其中的规则，通过取固定段的字节计算其宽高，或直接使用`ImageIO`相关的接口可以获取其宽高，参考：

```objective-c
+ (CGSize)getImageSize:(NSData *)imageData {
    CGImageSourceRef source = CGImageSourceCreateWithData((__bridge CFDataRef)imageData, NULL);
    CGImageRef imageRef = CGImageSourceCreateImageAtIndex(source, 0, NULL);
    CGFloat width = CGImageGetWidth(imageRef);
    CGFloat height = CGImageGetHeight(imageRef);
    CFRelease(imageRef);
    CFRelease(source);
    return CGSizeMake(width, height);
}
```

### 3.4 细节优化

在初步走通了流程，验证了方案的可行性后，我们开始完善设计整套流程，并且测试其性能消耗。如测试发现从`storyboard`生成截图较为耗时，为此我们做了一个缓存策略，避免每次都去截图。

优化后完整流程图如下：

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-04-12-142538.png)

### 3.5 方案小结

经历了整个方案从调研到开发完成，以及多机型多系统的测试，我们对缓存启动图在不同系统版本上的表现差异性做了个简单归纳：

1.  缓存路径： 
    
    ```shell
    iOS13.0 及以上：Library/SplashBoard/Snapshots/${PRODUCT_BUNDLE_IDENTIFIER} - {DEFAULT GROUP}；
    iOS13.0 以下：Library/Caches/Snapshots/${PRODUCT_BUNDLE_IDENTIFIER}；
    ```
    
2. 图片格式： 
    iOS10.0 及以上：`KTX`
    iOS10.0 以下：`PNG`。

3. 系统缓存图目录读写权限：
    iOS10.0 及以上：有权限；
    iOS10.0 以下：无权限。

**四、总结**
--------

本方案主要用于解决启动图无法渲染、不更新等异常问题，能够让应用自动恢复正常的启动图，从用户角度来说最坏的情况是首次启动时展示了异常的启动图，但下次冷启时即可展示正常的启动图了，保证了用户体验。

理论上在本方案基础之上还可升级添加更多产品策略，但这里也忠告大家请勿滥用，并且未来苹果可能会修改该系统机制。

希望本文能够对碰到此类问题的同学们有所帮助，也欢迎大家对本文指正不足。

最后给大家奉上苹果爸爸关于启动图的官方文档，其中一段：

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-04-12-142802.png)

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2021-04-12-142806.png)  
呃。。。还是希望苹果爸爸能够 “完善” 这个问题。