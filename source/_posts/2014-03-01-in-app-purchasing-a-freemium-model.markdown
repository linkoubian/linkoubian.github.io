---
layout: post
title: "移动应用的Freemium模式"
date: 2014-03-01 20:12:22 +0800
comments: true
categories: iOS IAP
---
In-App Purchasing是一种目前比较常见的定价策略。软件可以免费安装并使用，但如果需要更多或更高级的功能则需要付费，即[Freemium](http://en.wikipedia.org/wiki/Freemium)模式。
<!--more-->
昨天给APP集成了[广告服务](http://balloonsys.com/blog/2014/02/28/make-money-from-ad/)。碰到有洁癖的用户，被抱怨是免不了的。所以今天抓紧学习IAP相关的开发技术后，又马不停蹄的增加了付费去广告的功能。

## 创建IAP商品
用户只需为“去除广告”这个服务付费一次，并可永久性的免除广告的烦扰。所以这种情况下，“去除广告”应被看成Non-Consumable商品。而游戏中常见的“金币”是用完还需重新购买，属于快速消耗品，对应iTunes Connect后台里的Consumable商品。

商品的ID可采用APP_ID.PRODUCT_NAME的格式，该ID将在编码时用到。

注意创建IAP商品时遇到的Hosting Content with Apple这一项，此处选择No。关于这个Hosting Content的用法，可以参考这篇[文章](http://www.techotopia.com/index.php/Configuring_and_Creating_App_Store_Hosted_Content_for_iOS_6_In-App_Purchases)。

添加好名为“remove_ads”的商品后，该IAP的状态为“Ready to Submit”。同时系统提示我说“你的首个IAP必须与新版APP一起提交，请在Version Detail页面选中相关的IAP后点击准备上传按钮”。

收到这样的提示，略有惊喜。因为添加IAP时，该APP有个update还在等待review，而IAP是下个release的内容，当时有点担心此次的操作会影响等待审核的版本。结果证明担心是多余的。

## 设置工程
在构建目标的Capabilities页面，开启In-App Purchase功能。Xcode会自动为当前的App ID增加相关权限，同时链接StoreKit框架。

IAP所涉及到的交互流程主要有如下几步，  
1. APP首先向苹果服务器发起请求（含有商品ID集合）以获得商品信息  
2. 客户端以自定义的方式展示返回的商品  
3. 用户选中某个商品，触发支付请求，请求被加到支付队列  
4. 支付被苹果处理后，会发出Transaction Updated事件  
5. 客户端设置的交易监听者根据Transaction的状态执行不同逻辑  

收到Transaction Updated事件后，无论Transaction的状态是completed、failed还是restore，都需要调用finishTransaction以通知苹果服务器相关的通知已收到，否则会不停的收到Transaction Updated事件。

## 开始编码
### 请求商品信息
``` objc
	self.productsRequest = [[SKProductsRequest alloc] initWithProductIdentifiers:self.productIdentifiers];
	self.productsRequest.delegate = self;
	[self.productsRequest start];
```

当苹果服务器做出“成功”的响应时，会调用SKProductsRequestDelegate方法，从response里我们可以拿到products信息。

### 展示返回的商品
目前采用的是在“关于”页面，放一个静态的table view cell，用户点击后该cell后弹出ActionSheet，让用户选择是首次“购买”还是“恢复”曾经的交易。

看似这个过程和上一步无关，这是因为此处只关心一个“Remove Ads”商品，也不需要显示服务端设定的商品价格等信息。其实此处ActionSheet所显示的选项内容，完全可以由之前获得的product信息构成。

### 选中商品，触发支付请求
``` objc
	SKPayment *payment = [SKPayment paymentWithProduct:product];
	[[SKPaymentQueue defaultQueue] addPayment:payment];
```

支付请求仅需被加到支付队列即可，StoreKit会完成剩余的工作。当交易的状态发生变化时，客户端APP会收到Transaction Updated事件。

### 监听交易状态变化
``` objc
- (void)paymentQueue:(SKPaymentQueue *)queue updatedTransactions:(NSArray *)transactions
{
	for (SKPaymentTransaction *transaction in transactions){
		switch (transaction.transactionState) {
			case SKPaymentTransactionStatePurchased:
				[self completeTransaction:transaction];
				break;
			case SKPaymentTransactionStateFailed:
				[self failedTransaction:transaction];
				break;
			case SKPaymentTransactionStateRestored:
				[self restoreTransaction:transaction];
				break;
			default:
				break;
		}
	}
}
```

### 处理商品交易
上面用到的completeTransaction等方法，一方面根据“交易”的状态去设置User Defaults中对应商品的状态（是否已购买）；另一方面调用finishTransaction以通知苹果服务器相关的通知已收到。

示例代码如下，
``` objc
- (void)completeTransaction:(SKPaymentTransaction *)transaction
{
	// Cache purchased product.
	[self.purchasedProductIdentifiers addObject:productIdentifier];

	// Persist purchased product.
	[[NSUserDefaults standardUserDefaults] setBool:YES forKey:productIdentifier];
	[[NSUserDefaults standardUserDefaults] synchronize];

	// Notify product purchased, so then UI can be updated accordingly
	[[NSNotificationCenter defaultCenter] postNotificationName:ProductPurchasedNotification object:productIdentifier];

	// Removes transaction from the queue.
	[[SKPaymentQueue defaultQueue] finishTransaction:transaction];
}
```

其中ProductPurchasedNotification是一个自定义消息，收到该消息的控制器并可以做出适当的响应，比如隐藏掉购买按钮or修改按钮的文字or弹框提示用户“购买成功”...

## 问题分析
从打算加入一个IAP到跑通代码所需要的“必要劳动时间”，远超最初的估计。期间可能遇到的各种错误及相应地应对建议，可以参考[OneV's Den](http://www.weibo.com/onevcat)的[文章](http://onevcat.com/2013/11/ios-iap-checklist/)。

## 功能测试
IAP的测试，需要在iTunes Connect后台建立一个测试账号。从网络上获得的一点建议：使用之前最好退出平常使用的正式账号，另外也不要用测试账号去下载APP。原因不详，也不想以身试法，遵守就是了！

## 参考资料
如果想系统并深入的学习IAP的开发技术，可以阅读下列文章，    
1. [Ray Wenderlich](http://www.raywenderlich.com/u/rwenderlich)提供的[IAP入门教程](http://www.raywenderlich.com/21081/introduction-to-in-app-purchases-in-ios-6-tutorial)  
2. [Ray Wenderlich](http://www.raywenderlich.com/u/rwenderlich)提供的[IAP进阶教程](http://www.raywenderlich.com/23266/in-app-purchases-in-ios-6-tutorial-consumables-and-receipt-validation)  
3. 苹果官方提供的开发[文档](https://developer.apple.com/in-app-purchase/)  
 
如果你需要快速了解IAP的话，不妨读读[Brian Love](http://brianflove.com/author/blove/)的这篇[文章](http://brianflove.com/2013/02/11/in-app-purchasing-in-ios-6/)。
如果你需要快速实现IAP的话，不妨试试[Mattt Thompson](http://mattt.me)的开源库[CargoBay](https://github.com/mattt/CargoBay)。