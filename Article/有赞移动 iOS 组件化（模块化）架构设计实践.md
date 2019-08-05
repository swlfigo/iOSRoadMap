> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 https://juejin.im/post/5d4136295188255d5861d0e4

一、背景
----

业务组件化（或者叫模块化）作为移动端应用架构的主流方式之一，近年来一直是业界积极探索和实践的方向。有赞移动团队自 16 年起也在不断尝试各种组件化方案，在有赞微信商城，有赞零售，有赞美业等多个应用中进行了实践。我们踩过一些坑，也收获了很多宝贵的经验，并沉淀出 iOS 相关框架 [Bifrost](https://link.juejin.im?target=https%3A%2F%2Fgithub.com%2Fyouzan%2FBifrost) (雷神里的彩虹桥)。在过程中我们深刻体会到 “没有绝对正确的架构，只有最合适的架构” 这句话的意义。很多通用方案只是组件化的冰山一角，实际落地过程中还有相当多的东西需要考量。  
本文并不准备对组件化架构设计方案给出一份标准答案，而是希望通过我们的实践经验和思考分析，提供一种思路，对遇到类似问题的同学能有所启发。

**注**：

1.  区别于功能模块 / 组件（比如图片库，网络库），本文讨论的是业务模块 / 组件（比如订单模块，商品模块）相关的架构设计。
2.  相比组件（Component），个人感觉称之为模块（Module）更为合适。组件强调物理拆分，以便复用；模块强调逻辑拆分，以便解耦。而且如果用过 Android Studio, 会发现它创建的子系统都叫 Module. 但介于业界习惯称之为组件化，所以我们继续使用这个术语。本文下面所用名词，“模块” 等同于 “组件”。

二、什么是业务模块化（组件化）
---------------

传统的 App 架构设计更多强调的是分层，基于设计模式六大原则之一的单一职责原则，将系统划分为基础层，网络层，UI 层等等，以便于维护和扩展。但随着业务的发展，系统变得越来越复杂，只做分层就不够了。App 内各子系统之间耦合严重, 边界越来越模糊，经常发生你中有我我中有你的情况（图一）。这对代码质量，功能扩展，以及开发效率都会造成很大的影响。此时，一般会将各个子系统划分为相对独立的模块，通过中介者模式收敛交互代码，把模块间交互部分进行集中封装, 所有模块间调用均通过中介者来做（图二）。这时架构逻辑会清晰很多，但因为中介者仍然需要反向依赖业务模块，这并没有从根本上解除循坏依赖等问题。时不时发生一个模块进行改动，多个模块受影响编译不过的情况。进一步的，通过技术手段，消除中介者对业务模块依赖，即形成了业务模块化架构设计（图三）。

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-08-05-094932.jpg)

通过业务模块化架构，一般可以达到明确模块职责及边界，提升代码质量，减少复杂依赖，优化编译速度，提升开发效率等效果。很多文章都有相关分析，在此不再累述。

三、业界常见模块化方案
-----------

业务模块化设计通过对各业务模块的解耦改造，避免循环双向依赖，达到提升开发效率和质量的目的。但业务需求的依赖是无法消除的，所以模块化方案首先要解决的是如何在无代码依赖的情况下实现跨模块通信的问题。iOS 因为其强大的运行时特性，无论是基于 NSInvocation 还是基于 peformSelector 方法, 都可以很很容易做到这一点。但不能为了解耦而解耦，提升质量与效率才是我们的目的。直接基于 hardcode 字符串 + 反射的代码明显会极大损害开发质量与效率，与目标背道而驰。所以，模块化解耦需求的更准确的描述应该是 “如何在保证开发质量和效率的前提下做到无代码依赖的跨模块通信”。  
目前业界常见的模块间通讯方案大致如下几种：

1.  基于路由 URL 的 UI 页面统跳管理。
2.  基于反射的远程接口调用封装。
3.  基于面向协议思想的服务注册方案。
4.  基于通知的广播方案。

根据具体业务和需求的不同，大部分公司会采用以上一种或者某几种的组合。

### 3.1 路由 URL 统跳方案

统跳路由是页面解耦的最常见方式，大量应用于前端页面。通过把一个 URL 与一个页面绑定，需要时通过 URL 可以方便的打开相应页面。

```
//通过路由URL跳转到商品列表页面
//kRouteGoodsList = @"//goods/goods_list"
UIViewController *vc = [Router handleURL:kRouteGoodsList]; 
if(vc) {
    [self.navigationController pushViewController:vc animated:YES];
}
复制代码

```

当然有些场景会比这个复杂，比如有些页面需要更多参数。 基本类型的参数，URL 协议天然支持：

```
//kRouteGoodsDetails = @“//goods/goods_detail?goods_id=%d”
NSString *urlStr = [NSString stringWithFormat:@"kRouteGoodsDetails", 123];
UIViewController *vc = [Router handleURL:urlStr];
if(vc) {
   [self.navigationController pushViewController:vc animated:YES];
}
复制代码

```

复杂类型的参数，可以提供一个额外的字典参数 complexParams, 将复杂参数放到字典中即可：

```
+ (nullable id)handleURL:(nonnull NSString *)urlStr
           complexParams:(nullable NSDictionary*)complexParams
              completion:(nullable RouteCompletion)completion;
复制代码

```

上面方法里的 completion 参数，是一个回调 block, 处理打开某个页面需要有回调功能的场景。比如打开会员选择页面，搜索会员，搜到之后点击确定，回传会员数据：

```
//kRouteMemberSearch = @“//member/member_search”
UIViewController *vc = [Router handleURL:urlStr complexParams:nil completion:^(id  _Nullable result) {
    //code to handle the result
    ...
}];
if(vc) {
    [self.navigationController pushViewController:vc animated:YES];
}
复制代码

```

考虑到实现的灵活性，提供路由服务的页面，会将 URL 与一个 block 相绑定。block 中放入所需的初始化代码。可以在合适的地方将初始化 block 与路由 URL 绑定，比如在 +load 方法里：

```
+ (void)load {
    [Router bindURL:kRouteGoodsList
           toHandler:^id _Nullable(NSDictionary * _Nullable parameters) {
        return [[GoodsListViewController alloc] init];
    }];
}
复制代码

```

更多路由 URL 相关例子，可以参考 [Bifrost](https://link.juejin.im?target=https%3A%2F%2Fgithub.com%2Fyouzan%2FBifrost) 项目中的 Demo.

URL 本身是一种跨多端的通用协议。使用路由 URL 统跳方案的优势是动态性及多端统一 (H5, iOS，Android，Weex/RN); 缺点是能处理的交互场景偏简单。所以一般更适用于简单 UI 页面跳转。一些复杂操作和数据传输，虽然也可以通过此方式实现，但都不是很效率。 目前天猫和蘑菇街都有使用路由 URL 作为自己的页面统跳方案，达到解耦的目的。

### 3.2 基于反射的远程调用封装

当无法 import 某个类的头文件但仍需调用其方法时，最常想到的就是基于反射来实现了。例：

```
Class manager = NSClassFromString(@"YZGoodsManager");
NSArray *list = [manager performSelector:@selector(getGoodsList)];
//code to handle the list
...
复制代码

```

但这种方式存在大量的 hardcode 字符串。无法触发代码自动补全，容易出现拼写错误，而且这类错误只能在运行时触发相关方法后才能发现。无论是开发效率还是开发质量都有较大的影响。

如何进行优化呢？这其实是各端远程调用都需要解决的问题。移动端最常见的远程调用就是向后端接口发网络请求。针对这类问题，我们很容易想到创建一个网络层，将这类 “危险代码” 封装到里面。上层业务调用时网络层接口时，不需要 hardcode 字符串，也不需要理解内部麻烦的逻辑。

类似的，我可以将模块间通讯也封装到一个 “网络层” 中（或者叫消息转发层）。这样危险代码只存在某几个文件里，可以特别地进行 code review 和联调测试。后期还可以通过单元测试来保障质量。模块化方案中，我们可以称这类 “转发层” 为 Mediator (当然你也可以起个别的名字)。同时因为 performSelector 方法附带参数数量有限，也没有返回值，所以更适合使用 NSInvocation 来实现。

```
//Mediator提供基于NSInvocation的远程接口调用方法的统一封装
- (id)performTarget:(NSString *)targetName
             action:(NSString *)actionName
             params:(NSDictionary *)params;

//Goods模块所有对外提供的方法封装在一个Category中
@interface Mediator(Goods)
- (NSArray*)goods_getGoodsList;
- (NSInteger)goods_getGoodsCount;
...
@end
@impletation Mediator(Goods)
- (NSArray*)goods_getGoodsList {
    return [self performTarget:@“GoodsModule” action:@"getGoodsList" params:nil];
}
- (NSInteger)goods_getGoodsCount {
    return [self performTarget:@“GoodsModule” action:@"getGoodsCount" params:nil];
}
...
@end
复制代码

```

然后各个业务模块依赖 Mediator, 就可以直接调用这些方法了。

```
//业务方依赖Mediator模块，可以直接调用相关方法
...
NSArray *list = [[Mediator sharedInstance] goods_getGoodsList];
...
复制代码

```

这种方案的优势是调用简单方便，代码自动补全和编译时检查都仍然有效。 劣势是 category 存在重名覆盖的风险，需要通过开发规范以及一些检查机制来规避。同时 Mediator 只是收敛了 hardcode, 并未消除 hardcode, 仍然对开发效率有一定影响。

业界的 CTMediator 开源库，以及美团都是采用类似方案。

### 3.3 服务注册方案

有没有办法绝对的避免 hardcode 呢？如果接触过后端的服务化改造，会发现和移动端的业务模块化很相似。Dubbo 就是服务化的经典框架之一。它是通过服务注册的方式来实现远程接口调用的。即每个模块提供自己对外服务的协议声明，然后将此声明注册到中间层。调用方能从中间层看到存在哪些服务接口，然后直接调用即可。例：

```
//Goods模块提供的所有对外服务都放在GoodsModuleService中
@protocol GoodsModuleService
- (NSArray*)getGoodsList;
- (NSInteger)getGoodsCount;
...
@end
//Goods模块提供实现GoodsModuleService的对象, 
//并在+load方法中注册
@interface GoodsModule : NSObject<GoodsModuleService>
@end
@implementation GoodsModule
+ (void)load {
    //注册服务
    [ServiceManager registerService:@protocol(service_protocol) 
                  withModule:self.class]
}
//提供具体实现
- (NSArray*)getGoodsList {...}
- (NSInteger)getGoodsCount {...}
@end

//将GoodsModuleService放在某个公共模块中，对所有业务模块可见
//业务模块可以直接调用相关接口
...
id<GoodsModuleService> module = [ServiceManager objByService:@protocol(GoodsModuleService)];
NSArray *list = [module getGoodsList];
...
复制代码

```

这种方式的优势也包括调用简单方便。代码自动补全和编译时检查都有效。实现起来也简单，协议的所有实现仍然在模块内部，所以不需要写反射代码了。同时对外暴露的只有协议，符合团队协作的 “面向协议编程” 的思想。劣势是如果服务提供方和使用方依赖的是公共模块中的同一份协议（protocol）, 当协议内容改变时，会存在所有服务依赖模块编译失败的风险。同时需要一个注册过程，将 Protocol 协议与具体实现绑定起来。

业界里，蘑菇街的 ServiceManager 和阿里的 BeeHive 都是采用的这个方案。

### 3.4 通知广播方案

基于通知的模块间通讯方案，实现思路非常简单, 直接基于系统的 NSNotificationCenter 即可。 优势是实现简单，非常适合处理一对多的通讯场景。 劣势是仅适用于简单通讯场景。复杂数据传输，同步调用等方式都不太方便。 模块化通讯方案中，更多的是把通知方案作为以上几种方案的补充。

### 3.5 其它

除了模块间通讯的实现，业务模块化架构还需要考虑每个模块内部的设计，比如其生命周期控制，复杂对象传输，重复资源的处理等。可能因为每个公司都有自己的实际场景，业界方案里对这些问题描述的并不是很多。但实际上他们非常重要，有赞在模块化过程中做了很多相关思考和尝试，会在后面环节进行介绍。

四、有赞的模块化实践
----------

有赞移动自 16 年起开始实践业务模块化架构方式，大致经历了 2016 年的尝试 + 摸索，2017 年的思考 + 优化以及 2018 年的成熟 + 沉淀几个阶段。期间有过对已有 App 的模块化改造，也试过直接应用于新起项目。模块化方案经历过几次改版，踩过一些坑，也收获了很多宝贵的经验。

### 4.1 v1.0: 尝试 + 摸索

16 年，有赞微信商城、有赞收银等 App 经历了初期的功能快速迭代，内部依赖混乱，耦合严重，急需优化重构。传统的 MVVM、MVP 等优化方式无法从全局层面解决这些问题。后来在 InfoQ 的 "移动开发前线" 微信群里听了蘑菇街的组件化方案分享，非常受启发。不过当时还是有一些顾虑，比如微信商城和收银当时都属于中小型项目，每端开发人员都只有 4-6 人。业务模块化改造后会形成一定的开发门槛，带来一定的开发效率下降。小项目适合模块化改造吗？其收益是否能匹配付出呢？但考虑到当时 App 各模块边界已经稳定，即使模块化改造出现问题，也可以用很小的代价将其降级到传统的中介者模式，所以改造开始了。

#### 4.1.1 模块间通信方式设计

首先是梳理我们的模块间通信需求，主要包括以下三种：

1.  **UI 页面跳转**。比如 IM 模块点击用户头像打开会员模块的用户详情页。
2.  **动作执行及复杂数据传输**。比如商品模块向开单模块传递商品数据模型并进行价格计算。
3.  **一对多的通知广播**。比如 logout 时账号模块发出广播，各业务模块进行 cache 清理及其它相应操作。

我们选择了路由 URL + 远程接口调用封装 + 广播相结合的方式。

对于远程接口调用的封装方式，我们没有完全照抄 Mediator 方案。当时非常期望保留模块化的编译隔离属性。比如当 A 模块对外提供的某个接口发生变化时，不会引发依赖这个接口的模块的编译错误。这样可以避免依赖模块被迫中断手头的工作先去解决编译问题。当时也没有采用 Beehive 的服务注册方式，也是因为同样的原因。 经过讨论，当时选择参考网络层封装方式，在每个模块中设计一个对外的 “网络层” ModuleService。将对其它模块的接口的反射调用，放入各个模块的 ModuleService 中。 同时，我们希望各业务模块不需要去理解所依赖模块的内部复杂实现。比如 A 模块依赖 D 模块的 class D1 的接口 method1， class D2 的接口 method2, class D3 的接口 method3. A 需要了解 D 模块的这些内部信息才能完成反射功能的实现。如果 D 模块中这些命名有所变化，还会出现调用失败。所以我们对各个模块使用外观（Facade）模式进行重构。D 模块创建一个外观层 FacadeD. 通过 FacadeD 对象对外提供所有服务，同时隐藏内部复杂实现。调用方也只需要理解 FacadeD 的头文件 包含哪些接口即可。

**外观（Facade）模式**: 为子系统中的一组接口提供一个一致的界面， Facade 模式定义了一个高层接口，这个接口使得这一子系统更加容易使用。引入外观角色之后，用户只需要直接与外观角色交互，用户与子系统之间的复杂关系由外观角色来实现，从而降低了系统的耦合度。

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-08-05-095011.jpg)

另外，为什么还需要路由 URL 呢？ 其实从功能角度，远程接口的网络层，完全可以取代路由 URL 实现页面跳转，而且没有路由 URL 的一些 hardcode 的问题。而且路由 URL 和 远程接口存在一定的功能重合，还会造成后续实现新功能时，分不清应选择路由 URL 还是选择远程接口的困惑。这里选择支持路由 URL 的主要原因是我们存在动态化且多端统一的需求。比如消息模块下发的各种消息数据模型完全是动态的。后端配好展示内容以及跳转需求后，客户端不需要理解具体需求，只需要通过统一的路由跳转协议执行跳转动作即可。

#### 4.1.2 模块内设计及 App 结构调整

每个模块除了 Facade 模式改造之外，还需要考虑以下问题：

1.  合适的注册及初始化方式。
2.  接收并处理全局事件。
3.  App 层和 Common 层设计。
4.  模块编译产出以及集成到 App 中的方式。

因为考虑到每个 App 中业务模块数量不会很多（我们几个 App 内大多是 20 个左右），所以我们为每个模块创建了一个 Module 对象并令其为单例。在 +load 方法中将自身注册给模块化 SDK [Bifrost](https://link.juejin.im?target=https%3A%2F%2Fgithub.com%2Fyouzan%2FBifrost). 经测试，这里因为单例造成的内存占用以及 +load 方法引起的启动速度影响都微乎其微。模块需要监听的全局事件主要为 UIApplicationDelegate 中的那些方法。所以我们定义了一个继承 UIApplicationDelegate 的协议 BifrostModuleProtocol，令每个模块的 Module 对象都服从这个协议。App 的 AppDelegate 对象，会轮询所有注册了的业务模块并进行必要的调用。

```
@protocol BifrostModuleProtocol <UIApplicationDelegate, NSObject>
@required
+ (instancetype)sharedInstance;
- (void)setup;
...
@optional
+ (BOOL)setupModuleSynchronously;
...
@end
复制代码

```

所有业务代码挪入各业务模块的 Module 对象后，AppDelegate 非常干净。

```
@implementation YZAppDelegate
- (BOOL)application:(UIApplication *)application willFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    [Bifrost setupAllModules];
    [Bifrost checkAllModulesWithSelector:_cmd arguments:@[Safe(application), Safe(launchOptions)]];
    return YES;
}
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    [Bifrost checkAllModulesWithSelector:_cmd arguments:@[Safe(application), Safe(launchOptions)]];
    return YES;
}
- (void)applicationWillEnterForeground:(UIApplication *)application {
    [Bifrost checkAllModulesWithSelector:_cmd arguments:@[Safe(application)]];
}
...
@end
复制代码

```

每个业务模块都作为一个子 Project 集成入 App Project. 同时创建一个特殊的模块 Common，用于放置一些通用业务和全局的基类。App 层只保留 AppDelegate 等全局类和 plist 等特殊配置，基本没有任何业务代码。Common 层因为没有明确的业务组来负责，所以也应该尽量轻薄。各业务模块之间互不可见，但可以直接依赖 Common 模块。通过 search path 来设置模块依赖关系。

每个业务模块的产出包括可执行文件和资源文件两部分。有 2 种选择：生成 framework 和生成静态库 + 资源 bundle.

使用 framework 的优点是输出在同一个对象内，方便管理。缺点是作为动态库载入，影响加载速度。所以当时选择了静态库 + bundle 的形式。不过个人感觉这块还是需要具体测一下会慢做少再做决定更合适。但因为二者差别不大，所以后续我们也一直没作调整。

另外如果使用 framework，需要注意资源读取的问题。因为传统的资源读取方式无法定位到 framework 内资源，需要通过 bundleForClass: 才行。

```
//传统方式只能定位到指定bundle，比如main bundle中资源
NSURL *path = [[NSBundle mainBundle] URLForResource:@"file_name" withExtension:@"txt"]; 

// framework bundle需要通过bundleForClass获取
NSBundle *bundle = [NSBundle bundleForClass:classA]; //classA为framework中的某各类
// 读UIStoryboard
UIStoryboard *storyboard = [UIStoryboard storyboardWithName:@“sb_name” bundle:bundle];
// 读UIImage
UIImage *image = [UIImage imageNamed:@"icon_name" inBundle:bundle compatibleWithTraitCollection:nil];
...
复制代码

```

#### 4.1.3 复杂对象传输

当时最纠结的点就是复杂对象的传输。例如商品模型，它包含几十个字段。如果是传字典或传 json, 那么数据提供方（商品模块）和使用方（开单模块）都需要专门理解并实现一下这种模型的各种字段，对开发效率影响很大.

有没有办法直接传递模型对象呢？这里涉及到模型的类文件放在哪里。最容易想到的方案是沉入 Common 模块。但一旦这个口子放开，后续会有越来越多的模型放入 Common，和前面提到的简化 Common 层的目标是相悖的。而且因为 Common 模块没有明确业务组归属，所有小组都能编辑, 其质量和稳定性难以保障。最终我们采用了一个 tricky 的方案，把要传递的复杂模型的代码复制一份放在使用方模块中，同时通过修改类名前缀加以区分，这样就可以避免打包时的链接冲突错误。比如商品模块内叫 _YZG_GoodsModel, 开单模块内叫 _YZS_GoodsModel. 商品模块的接口返回的是 YZGGoodsModel，开单模块将其强转为 YZSGoodsModel 即可。

```
//YZSaleModuleService.m内
#import "YZSGoodsModel.h"

- (YZSGoodsModel*)goodsById:(NSString*)goodsId {
    //Sale Module远程调用Goods Module的接口
    id obj = [Bifrost performTarget:@"YZGoodsModule"
                           action:@"goodsById:"
                             params:@[goodsId]];
    //做一次强转
    YZSGoodsModel *goods = (YZSGoodsModel*)obj;
    return goods;
}
复制代码

```

这种方式虽然比较粗暴，但考虑到两个模块间交互的复杂对象应该不会很多（如果太多则应考虑这两个模块是否划分合适），同时拷贝粘贴操作起来成本可控，所以可以接受。同时这种方法也能达到预期的编译隔离的效果。但两边模型定义及实现还是有不一致的风险。为了解决一致性问题，我们做了个检查脚本工具，在编译时触发。会根据命名规则查找这类 “同名” model 的代码，并做一个比较。如果发现不一致，则报 warning. 注意不是报 error, 因为我们希望一个模块做了接口修改，另一个模块可以存在一种选择，是马上更新接口，还是先完成手头的工作将来再更新。

#### 4.1.4 重复资源处理

这类资源主要包括图片、音视频，数据模型等等。

首先我们排除了无脑放入 Common 的方案。因为下沉入 Common 会破坏各业务模块的完整性，同时也会影响 Common 的质量。经过讨论后，决定把资源分为三类：

1.  通用功能所用资源，将相关代码整理为功能组件后一起放入 Common.
2.  业务功能的大部分资源可以通过无损压缩控制体积，体积不大的资源允许一定程度上的重复。
3.  较大体积的资源放到服务端，App 端动态拉取放在本地缓存中。

同时平时定期通过自动化工具检测无用资源，以及重复资源的大小，以便及时优化包体积。

#### 4.1.5 体验与成果

基于以上设计，我们大概花了 3 的个月的时间对已有项目进行了业务模块化改造（边做业务边改造）。因为方案细节考虑的比较多，大家对一些可能存在的问题也都有预期，所以当时改造后大家多持肯定态度，成本 vs 收益还是可观的。

v1.0 版本改造后，App 架构关系如图：

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-08-05-095030.jpg)

App 项目结构如图：

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-08-05-095056.jpg)

### 4.2 v2.0: 思考 + 优化

16 年的第一版模块化设计方案虽然可行，但还存在两个痛点：

1.  模块间网络层的封装基于反射代码, 写起来仍然有些麻烦。而且需要额外写单测保证质量。
2.  复杂对象的处理方式也存在一些问题，比如拷贝粘贴的方式比较丑陋，重复代码会带来包体积的增加。

上述问题在团队规模扩大，新同学到来时格外明显，经常需要答疑讲解。甚至有一次业务项目时间特别紧张时，有些小伙伴私下更改模块间头文件 search path，直接依赖的了别的模块，以便重用复杂模型类的情况。

这些问题的根本原因还是存在效率损失，"不方便"，怎么优化呢？

#### 4.2.1 远程接口封装优化

首先是如何避免反射及 hardcode. 阿里 Beehive 的基于服务注册的方式 是不需要 hardcode 代码的。但它有额外的服务注册过程，可能会影响启动速度，性能弱于基于反射的接口封装方案。这里对启动速度的影响究竟有多少呢？我们做了个测试，在 +load 方法中注册了 1000 个 Sevice Protocol, 启动时间影响大概是 2-4 ms, 非常少。

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-08-05-095114.jpg)

因为我们每个模块都是基于外观模式设计的。所以每个模块只需要对外暴露一个 Service Protocol 即可。我们 App 的实际模块数量大概是 20 个，所以对启动速度的影响可以忽略不计。而且前文提到，每个模块本来也需要注册自己的外观类（Module 对象）以处理生命周期和接受 AppDelegate 消息。这里 Service Protocl 的实现者就是这个 Module 对象，所以其实没有额外的性能消耗。

#### 4.2.2 复杂对象传输优化

之前的业务模块化方案没有使用 Beehive 还有个原因，就是服务提供方和使用方共同依赖同一个 Protocol，不符合我们编译隔离的需求。但既然我们可以拷贝粘贴复杂对象代码，是否也可以拷贝粘贴 Protocol 声明呢？答案是可行的。而且即使工程中同时存在多个同名的 Protocol 也不会引起编译问题，连改名这一步都省去了。以商品模型为例，为它定义一个 GoodModelProtocol, 服务使用方开单模块可以直接将这个 Protocol 的声明 copy 到自己模块中，也不需要改名，操作成本非常低。然后商品模块内就可以使用这个 Protocol 了。同时因为用的是同一个协议对象，所以 v1.0 中的类型强转风险也没有了。

跨模块进行方法调用和数据读取非常便捷：

```
NSString *goodsID = @"123123123";
id<YZGoodsModelProtocol> goods = [BFModule(YZGoodsModuleService) goodsById:goodsID];
self.goodsCell.name = goods.name;
self.goodsCell.price = goods.price;
...
复制代码

```

为尽量减少拷贝粘贴频率，我们将每个模块对外提供的接口服务，路由定义，通知定义，以及复杂对象 Protocol 定义都放在 ModuleService.h 中。管理非常方便规范，别的模块 copy 起来也简单，只需要把这个 ModuleService.h 文件 copy 到自己模块内部，就可以直接依赖并调用接口了。而且如果将来需要从服务器拉取相关配置，一个文件会方便很多。但是也需要考虑如果以上内容都放入同一个头文件，会不会导致文件过大的问题。当时分析模块间交互是有限的，否则就需要考虑模块划分是否合适。所以问题应该不大。从结果来看，目前我们最大的 ModuleService.h, 加上注释大概是 300 多行。

#### 4.2.3 其它优化

另外，我们发现每个模块对初始化顺序也有需求。比如账号模块的初始化可能要优先于别的模块，以便别的模块在初始化时使用其服务。所以我们也对 ModuleProtocol 增加了优先级接口。每个模块可以定义自己的初始化优先级。

```
/**
 The priority of the module to be setup. 0 is the lowest priority;
 If not provided, the default priority is BifrostModuleDefaultPriority;

 @return the priority
 */
+ (NSUInteger)priority;
复制代码

```

经过以上优化改造，基本解决了 v1.0 的所有质量及效率方面的隐患，业务模块化方案趋近成熟。

### 4.3 v3.0: 成熟 + 沉淀

17 年优化后的模块化方案，基本算是具有有赞特色的相对成熟的方案了，支撑了包括零售在内的多个大型 app 的开发。

#### 4.3.1 编译隔离的思考

Copy 头文件的方式仍然有一些理解成本。移动团队规模快速发展，一些新来的小伙伴还是会提出疑问。18 年年中我们做了几次检查，发现模块间 ModuleService 版本不一致的情况时有发生。当时零售移动团队虽然达到 30 多人，但仍然是一个协作紧密的整体，发版节奏基本一致。各业务模块代码都在同一个 git 工程中，基本每次发版用的都是各个模块的最新版本。而且实际做了几次调查，发现 ModuleService 中接口改变导致的依赖模块的修改，其实成本很低，改起来很快。此时我们开始思考之前追求的编译隔离是否适合当前阶段，是否有实际价值。

最终我们决定节省每一份精力，效率最大化。将各业务的 ModuleService 进行下沉到 Commom 模块，各业务模块直接依赖 Common 中的这些 ModuleServie 头文件，不再需要 copy 操作。这样改造的代价是形成了更多的依赖。本来一个业务模块是可以不依赖 Common 的，但现在就必须依赖了。但考虑到实际情况，还没有不依赖 Common 的业务模块存在，这种追求没有价值，所以应该问题不大。同时因为下沉的都是一些头文件，没有具体实现，将来如果需要模块间的进一步隔离，比如模块单独打包等，只需要将这些 Moduleservie 做到服务端可配置 + 自动化下载生成即可，改造成本非常小。

但这样改造后又发生了一件事。某个新来的同学，直接在 Common 模块中写代码通过这些 ModuleService 调用了上层业务模块的功能，形成了底层 Commmon 模块对上层业务模块的反向依赖。于是我们进一步拆分出了一个新模块 Mediator, 将 Bifrost SDK 和这些 ModuleSevice 放入其中。Common 模块和 Mediator 互不可见。

最终形成的 App 架构为：

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-08-05-095139.jpg)

**注**

：业界有些方案是把 ModuleServie 分开存放的，相当于把以上方案里的 Mediator 部分进行分拆，每个业务模块都有一个。这种方式的优点是职责明确，大家不用同时对一个公共模块进行修改，同时可以做到依赖关系很清晰；劣势是模块的数量增加了一倍，维护成本增加很多。考虑到我们目前的情况，Mediator 模块是很薄的一层，共同修改维护这个模块也可以接受，所以目前没有将其拆开。将来如果需要，再将其做分拆改造即可，改造工作量很小。

#### 4.3.2 代码隔离的思考

除了不在不合适的阶段追求编译隔离，我们还发现代码隔离并不适合我们。

业务模块化的效果之一就是个业务模块可以单独打包，放入壳工程运行。很容易想到的一个改造就是把各个模块拆到不同的 git 中。好处很多，比如单独的权限控制，独立的版本号，万一发版时发现问题可以及时 rollback 用老版本打包。我们的微信商城 App 就做了这种尝试。将代码迁到了很多 git 中，通过 pod 的方式进行管理。但后续开发中体验并不是很好。当时微信商城 App 的模块数量比开发同学数量多很多，每个同学都同时维护着多个模块。有时一个项目，一个人需要同时在多个 git 中修改多个模块的代码。修改完成后，要多次执行提交、打版本号以及集成测试等操作，很不效率。同时因为涉及到多个 git，代码提交的 Merge Request 和相关的编译检查也复杂了很多。同样的，因为微信商城 App 中不同模块的开发发版节奏也基本一致，所以多 git 多 pod 的不同版本管理及回退的优势也没有体现出来。最终还是将各模块代码迁回了主 git 中。

#### 4.3.3 没价值的隔离？

但编译隔离和代码隔离真的没有价值吗？当然不是，主要是我们当前阶段并不需要。过早的调整增加了成本却没有价值产出，所以并不合适。实际上我们还有一些业务模块是跨 App 使用的，比如 IM 模块，资产模块等等。他们都是独立 git 独立发版的。编译隔离和代码隔离属性对他们很有效。

另外，每个模块单独 git 可以有更细粒度的权限管理。我们因为在一个 git 中，曾发生过好几次小伙伴改别人的模块改出问题的例子（虽然有 MR, 但人难免有遗漏）。后来我们是通过 git commit hook + 修改文件路径来控制修改权限才解决了这个问题。后续介绍有赞移动基础设施建设的文章中会有更多相关细节。

#### 4.3.4 Bifrost (雷神里的彩虹桥)

最终，我们总结了所有我们需要的业务模块化需求，沉淀出了轻量级的模块化 SDK [Bifrost](https://link.juejin.im?target=https%3A%2F%2Fgithub.com%2Fyouzan%2FBifrost).

为什么不直接使用业界的 CTMediator 或者 Beehive 或者 MGJRouter, 要再造个轮子呢？主要有三个原因：一是我们开始尝试模块化改造时，业界还没有相关框架开源出来，所以需要自己实现。二是我们的需求和业界的开源库不完全相符。MGJRouter 缺少服务管理，CTMediator 和设计不符，Beehive 没有路由管理同时不够轻量 (很多接口还是基于阿里的需求提供的，我们用不到，会形成理解成本)。原因三其实是最关键的，就是模块化 SDK 的实现其实不难。通过前面的介绍，可以发现其中并没有什么黑魔法，代码量也不多，实现成本很低。模块化过程更多精力花在了全局架构设计，与之配合的开发规范，以及结合自己团队情况的一些取舍。模块化 SDK 只是模块化整体设计的冰山一角。我们也推荐读者所在团队，如果有时间可以尝试自己实现模块化工具，[Bifrost](https://link.juejin.im?target=https%3A%2F%2Fgithub.com%2Fyouzan%2FBifrost) 只用做参考即可。

#### 4.3.5 业务模块化时机

我们建议所有进入业务领域划分稳定期（业务模块基本确定，不会发生较大变动）的团队采用业务模块化架构设计。即使模块划分还没完全明确，也可以考虑对部分明确了模块进行模块化改造。因为迟早要用，晚用不如早用。目前基于路由 URL + 协议注册的模块间通讯方式，对开发效率基本无损。

五、总结
----

移动应用的业务模块化架构设计，其真正的目标是提升开发质量和效率。单从实现角度来看并没有什么黑魔法或技术难点，更多的是结合团队实际开发协作方式和业务场景的具体考量——“适合自己的才是最好的”。有赞移动团队通过过往 3 年的实践，发现一味的追求性能，绝对的追求模块间编译隔离，过早的追求模块代码管理隔离等方式都偏离了模块化设计的真正目的，是得不偿失的。更合适的方式是在可控的改造代价下，一定程度考虑未来的优化方式，更多的考虑当前的实际场景，来设计适合自己的模块化方式。希望通过本文提供的具体案例和思考方式，大家都能找到适合自己应用的业务模块化之路。