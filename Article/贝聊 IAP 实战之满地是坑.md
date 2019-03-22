> 原文地址 https://juejin.im/post/5a3b14f36fb9a045104aa6c8

> 注意: 文章中讨论的 IAP 是指使用苹果内购购买消耗性的项目。

这次为大家带来我司 IAP 的实现过程详解，鉴于支付功能的重要性以及复杂性，文章会很长，而且支付验证的细节也关系重大，所以这个主题会包含三篇。

> 第一篇：[[iOS] 贝聊 IAP 实战之满地是坑](https://juejin.im/post/5a3b14f36fb9a045104aa6c8)，这一篇是支付基础知识的讲解，主要会详细介绍 IAP，同时也会对比支付宝和微信支付，从而引出 IAP 的坑和注意点。
> 
> 第二篇：[[iOS] 贝聊 IAP 实战之见坑填坑](https://juejin.im/post/5a3b164e6fb9a0450e76466b)，这一篇是高潮性的一篇，主要针对第一篇文章中分析出的 IAP 的问题进行具体解决。
> 
> 第三篇：[[iOS] 贝聊 IAP 实战之订单绑定](https://juejin.im/post/5a3b169151882521033469b4)，这一篇是关键性的一篇，主要讲述作者探索将自己服务器生成的订单号绑定到 IAP 上的过程。

不用担心，我从来不会只讲原理不留源码，我已经将我司的源码整理出来，你使用时只需要拽到工程中就可以了，下面开始我们的内容 。

[源码在这里。](https://link.juejin.im?target=https%3A%2F%2Fgithub.com%2Fbeiliao-mobile%2FBLIAP)

## 01\. 题外话

今年上半年的公众号打赏事件，大家可还记得？我们对苹果强收过路费的行为愤懑，也为微信可惜不已，此事最后以腾讯高管团队访问苹果画上句号。显然，协商结果两位老板以及他们的团队都很满意。

## 02\. 熟悉的支付宝和微信支付

仔细看一下下面这张图，这是我们每次在买早餐使用支付宝支付的流程图。下面我们来一步一步看一下每一步对应的操作原理。

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-22-030039.jpg)

> **第一步**：我们的 APP 发起一笔支付交易，此时，第一件事，我们要去我们自己的服务器上创建一个订单信息。同时服务器会组装好一笔交易交给我们。关于组装交易信息，有两种做法，第一种就是支付宝推荐我们做的，由我们服务器来组装交易信息，服务器加密交易信息，并保存签名信息；另一种做法是，服务器返回商品信息给 APP，由 APP 来组装交易信息，并进行加密处理等操作。显然我们应该采用第一种方式。
> 
> **第二步**：服务器创建好交易信息以后，返回给 APP，APP 不对交易信息做处理。
> 
> **第三步**：APP 拿到交易信息，开始调起支付宝的 SDK，支付宝的 SDK 把交易信息传给支付宝的服务器。
> 
> **第四步**：验证通过以后，支付宝服务器会告诉支付宝 SDK 验证通过。
> 
> **第五步**：验证通过以后，我们的 APP 会调起支付宝 APP，跳转到支付宝 APP。
> 
> **第六步**：在支付宝 APP 里，用户输入密码进行交易，和支付宝服务器进行通讯。
> 
> **第七步**：支付成功，支付宝服务器回调支付宝 APP。
> 
> **第八步**：支付宝回到我们自己的 APP，并通过 `- (BOOL)application:(UIApplication *)application openURL:(NSURL *)url sourceApplication:(NSString *)sourceApplication annotation:(id)annotation` 方法处理支付宝的回调结果，对应的进行刷新 UI 等操作。
> 
> 第九步：支付宝服务器会回调我们的服务器并把收据传给我们服务器，如果我们的服务器没有确认已经收到支付宝的收据信息，那么支付宝服务器就会一直回调我们的服务器，只是回调时间间隔会越来越久。
> 
> **第十步**：我们的服务器收到支付宝的回调，并回调支付宝，确认已经收到收据信息，此时早餐买完了。

支付宝的支付流程讲完了，那微信支付也讲完了，因为它们流程相似。

## 03\. 坑爹的 IAP 支付

IAP 坑爹之处从以下两个方面来理解。

第一方面，APP 不接 IAP 审核不让过。接不接 IAP，苹果不是和你商量，而是强制要求，爸爸说怎么样，就怎么样。当然，这篇文章解决不了这个问题，所以也只是说说而已。上面说了微信公众号的事情，虽然它不是 IAP 的事情，但是实质上都属于强收过路费的行为。

第二方面，坑开发人员。下面开始数坑。

![](http://sylarimage.oss-cn-shenzhen.aliyuncs.com/2019-03-22-030102.jpg)

只有 8 步，比支付宝少 2 步，对不对？看起来比支付宝还简单，有木有？

> **第一步**：用户开始购买，首先会去我们自己的服务器创建一个交易订单，返回给 APP。
> 
> **第二步**：APP 拿到交易信息，然后开始调起 IAP 服务创建订单，并把订单推入支付队列。
> 
> **第三步**：IAP 会和 IAP 服务器通讯，让用户确认购买，输入密码。
> 
> **第四步**：IAP 服务器回调 APP，通知购买成功，并把收据写入到 APP 沙盒中。
> 
> **第五步**：此时，APP 应该去获取沙盒中的收据信息（一段 Base 64 编码的数据），并将收据信息上传给服务器。
> 
> **第六步**：服务器拿到收据以后，就应该去 IAP 服务器查询这个收据对应的已付款的订单号。
> 
> **第七步**：我们自己的服务器拿到这个收据对应的已付款的订单号以后，就去校验当前的已付款订单中是否有要查询的那一笔，如果有，就告诉 APP。
> 
> **第八步**：APP 拿到查询结果，然后把这笔交易给 finish 掉。

## 04\. 对比支付宝和 IAP

没啥大毛病，对吧？现在来详细分析一下。

由于移动端所处的网络环境远远比服务端要复杂，所以，最大可能出现问题的是与移动端的通讯上。对于支付宝，只要移动端确实付款完成，那么接下来的验证工作都是服务器于服务器之间的通讯。这样一来，只要用户确实产生了一笔交易，那么接下来的验证就变得可靠的多，而且支付宝服务器会一直回调我们的服务器，交易的可靠性得到了极大的保证。

同样，我们再来看看 IAP，交易是一样的。但是**验证交易这一环需要移动端来驱动我们自己的服务器来进行查询，这是第一个坑，先记一笔**。另外一点，**IAP 的服务器远在美国，我们的服务器去查询延时相当严重，这是其二**。

## 05.IAP 设计上的坑

上面讲了两个很大的坑，接下来看一看 IAP 本身有哪些坑。最大的一个就是，从 IAP 交易结果出来到通知 APP，只有一次。这里有以下几个问题：

> 1\. 如果用户后买成功以后，网络就不行了，那么苹果的 IAP 也收不到支付成功的通知，就没法通知 APP，我们也没法给用户发货。
> 
> 2\. 如果 IAP 通知我们支付成功，我们驱动服务器去 IAP 服务器查询失败的话，那就要等下次 APP 启动的时候，才会重新通知我们有未验证的订单。这个周期根本没法想象，如果用户一个月不重启 APP，那么我们可能一个月没法给用户发货。
> 
> 3\. 有人反馈，IAP 通知已经交易成功了，此时去沙盒里取收据数据，发现为空，或者出现通知交易成功那笔交易没有被及时的写入到沙盒数据中，导致我们服务器去 IAP 服务器查询的时候，查不到这笔订单。
> 
> 4\. 如果用户的交易还没有得到验证，就把 APP 给卸载了，以后要怎么恢复那些没有被验证的订单？
> 
> 5\. 越狱手机有无数奇葩的收据丢失或无效或被替换的问题，应该怎样酌情处理？
> 
> 6\. 交易没有发生变化，仅仅是重启一下，收据信息就会发生改变。
> 
> 7\. 当验证交易成功以后我们去取 IAP 的待验证交易列表的时候，这个列表没有数据。

好吧，算起来有九个比较大的问题了，还有没照顾到的请各位补充。这九个问题，基本上每一个都是致命的。这么多的不确定性，我们应该怎么综合处理，怎么相互平衡？

我们先放一放这些问题，下一篇就一起来着手解决这些问题，现在我们先来看一看 IAP 支付的基本代码。

## 06.IAP 支付代码

我们先不去想那么多，先把支付逻辑跑通再说。下面我们看看 IAP 的代码。

```objective-c
#import <StoreKit/StoreKit.h>

@interface BLPaymentManager ()<SKPaymentTransactionObserver, SKProductsRequestDelegate>

@end

@implementation BLPaymentManager

- (void)dealloc {
    [[SKPaymentQueue defaultQueue] removeTransactionObserver:self];
}

- (void)init {
    self = [super init];
    if(self) {
         [[SKPaymentQueue defaultQueue] addTransactionObserver:self];
    }
    return self;
}

- (void)buyProduction {
    if ([SKPaymentQueue canMakePayments]) {

        [self getProductInfo:nil];

    } else {
        NSLog(@"用户禁止应用内付费购买");
    }
}

// 从Apple查询用户点击购买的产品的信息.
- (void)getProductInfo:(NSString *)productIdentifier {
    NSSet *identifiers = [NSSet setWithObject:productIdentifier];
    SKProductsRequest *request = [[SKProductsRequest alloc] initWithProductIdentifiers:identifiers];
    request.delegate = self;
    [request start];
}

#pragma mark - SKPaymentTransactionObserver

// 购买操作后的回调.
- (void)paymentQueue:(SKPaymentQueue *)queue updatedTransactions:(NSArray<SKPaymentTransaction *> *)transactions {
    // 这里的事务包含之前没有完成的.
    for (SKPaymentTransaction *transcation in transactions) {
        switch (transcation.transactionState) {
            case SKPaymentTransactionStatePurchasing:
                [self transcationPurchasing:transcation];
                break;

            case SKPaymentTransactionStatePurchased:
                [self transcationPurchased:transcation];
                break;

            case SKPaymentTransactionStateFailed:
                [self transcationFailed:transcation];
                break;

            case SKPaymentTransactionStateRestored:
                [self transcationRestored:transcation];
                break;

            case SKPaymentTransactionStateDeferred:
                [self transcationDeferred:transcation];
                break;
        }
    }
}

#pragma mark - TranscationState

// 交易中.
- (void)transcationPurchasing:(SKPaymentTransaction *)transcation {
    NSURL *receiptURL = [[NSBundle mainBundle] appStoreReceiptURL];
    NSData *receipt = [NSData dataWithContentsOfURL:receiptURL];
    if (!receipt) {
        NSLog(@"没有收据, 处理异常");
        return;
    }
}

// 交易成功.
- (void)transcationPurchased:(SKPaymentTransaction *)transcation {
    // 存储到本地先.
    // 发送到服务器, 等待验证结果.
    [[SKPaymentQueue defaultQueue] finishTransaction:transcation];
}

// 交易失败.
- (void)transcationFailed:(SKPaymentTransaction *)transcation {

}

// 已经购买过该商品.
- (void)transcationRestored:(SKPaymentTransaction *)transcation {

}

// 交易延期.
- (void)transcationDeferred:(SKPaymentTransaction *)transcation {

}

#pragma mark - SKProductsRequestDelegate

// 查询成功后的回调.
- (void)productsRequest:(SKProductsRequest *)request didReceiveResponse:(SKProductsResponse *)response {
    NSArray<SKProduct *> *products = response.products;
    if (!products.count) {
        NSLog(@"没有正在出售的商品");
        return;
    }

    SKPayment *payment = [SKPayment paymentWithProduct:products.firstObject];
    [[SKPaymentQueue defaultQueue] addPayment:payment];
}

@end

```

代码大致做了如下事情，初始化的时候去添加支付结果的监听，并在 `-dealloc:` 方法中移除监听。同时可以通过 `- （void)fetchProductInfoWithProductIdentifiers:(NSSet<NSString *> *)productIdentifiers` 方法查询后台配置的商品信息。通过 `-buyProduction：` 方法购买产品，购买成功以后，IAP 通过 `- (void)paymentQueue:(SKPaymentQueue *)queue updatedTransactions:(NSArray<SKPaymentTransaction *> *)transactions` 方法通知购买进度。