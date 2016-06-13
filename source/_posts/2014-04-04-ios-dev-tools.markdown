---
layout: post
title: "做iOS外包开发用到的工具"
date: 2014-04-04 16:27:54 +0800
comments: true
categories: iOS, Tools
---
这个虚拟的Balloonsys工作室，总算发展到两位工程师了。
<!--more-->
## 发展历史
2010年8月第一次为成都一家公司做兼职开发，仅持续两周  
2012年9月帮杭州一家公司发布应用  
2014年4月终于签下了第一份APP开发合同

## 第三方服务
为自己、为别人开发iPhone应用的过程中，也使用过不少第三方服务。一直持续使用的有：  
1. 私有代码托管 https://bitbucket.org   
2. 收集崩溃日志 https://www.crashlytics.com   
3. 测试版本分发 https://testflightapp.com   
4. 数据统计分析 http://www.appannie.com   

## 软件工具
当一个关于APP的Idea萌生后，我会在草稿本上大致画出轮廓，想好每一页的交互，然后用下面的软件工具落地：   
1. 网络请求分析 http://www.portswigger.net/burp/download.html   
2. 软件原型设计 http://www.apple.com/cn/mac/keynote/   
3. 接口服务文档 http://www.apple.com/cn/mac/pages/   
4. 模拟接口实现 http://www.sublimetext.com   
5. 测试一下接口 http://luckymarmot.com/paw   
6. 买点现成图标 https://www.iconfinder.com   
7. 编辑矢量图标 http://www.bohemiancoding.com/sketch/   

在Mac上启动Burp后，把iPhone的代理服务器设置为MBPR的IP，端口为Burp中设置的那个。这样就可以在Burp中方便的看到手机上的网络请求详情。通过这种方式，我获得了不少第三方应用的API，然后开发了两款“非官方”的客户端。

利用Keynote做原型设计，这是从公司的咨询顾问那里学来的。感觉速度快，效果也很不错。在开发西子快讯前，我便用Keynote做了一份演示原型。

通常后端服务返回JSON格式的数据，为方便的观察API的Response，我喜欢用Paw这个工具，个人感觉比Post Man好用很多。

做客户端开发，对图标的需求还是蛮大的，按钮、菜单、启动图标等等。Icon Finder这个网站不光提供免费的图标下载，还有付费的图标，也不贵，而且提供SVG格式。利用Sketch，可以轻松修改颜色、大小等。

## 个人小结
荀子在劝学篇里说，“君子生非异也，善假于物也”！没错，做APP开发就是要善假于物也！