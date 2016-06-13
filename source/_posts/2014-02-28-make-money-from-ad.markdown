---
layout: post
title: "为应用添加广告支持"
date: 2014-02-28 16:22:38 +0800
comments: true
categories: iOS
---
年前做的一款关于公共自行车租赁的APP，自我感觉还是比较实用的。为了造福杭城更多的iPhone用户，新版本打算采用免费+iAP（去广告）的方式发行。所以，今天便研究了下iAd及AdMob两种广告平台的集成方法。
<!--more-->
## AdMob
想要支持AdMob，需要首先注册账号，添加应用，然后便可获得Publisher ID。至于如何配置工程以支持AdMob，可以参考[这里](https://developers.google.com/mobile-ads-sdk/docs/#ios)。

## iAd & AdMob
如果是手动编写代码以同时支持iAd和AdMob可参考[这里](http://www.apptite.be/tutorial_mixing_ads.php)。另外Github上有一个开源库[LARSAdController](https://github.com/larsacus/LARSAdController)，使用起来非常方便。

## 收益
集成AdMob的版本尚未上架，自己在手机上对着开发者证书安装的APP里的广告变着法子点击（换不同的VPN服务器、重置IDFA），竟然也能收获三十来刀。但这是冒着被封号的危险，谨慎啊！

这里有篇关于AdMob的[经验贴](http://blog.sina.com.cn/s/blog_7193dd920101keay.html)，很有料。

