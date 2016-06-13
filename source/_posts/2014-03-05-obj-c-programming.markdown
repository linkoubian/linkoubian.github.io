---
layout: post
title: "BNR出品的Obj-C教程"
date: 2014-03-05 21:22:44 +0800
comments: true
categories: 
---
据说The Big Nerd Ranch Guide系列教程都很不错，于是到[皮皮书屋](http://www.ppurl.com/2013/11/objective-c-programming-the-big-nerd-ranch-guide-second-edition-epub.html)下载了一份BNR出品的关于Obj-C的电子书。在iPad、Mac上阅读ePub格式的文档，有着很不错的体验。
<!--more-->
直接在电子书里记笔记的习惯还没有养成，依旧习惯于在[iA Writer](http://www.iawriter.com/mac/)里记录一些知识点以备日后查看。

1. 在C的函数里使用BOOL类型，需要#include <objc/objc.h>
2. sleep函数是在unistd.h头文件中声明的
3. EXIT_SUCCESS等程序退出标志，是在stdlib.h中声明的
4. 全局变量可以在多个文件中访问到；静态变量仅当前文件可以访问  
5. NSInteger在32位系统上占32位，在64位系统上占64位  
6. 在格式化输出NSInteger时，需强类型转换成long，然后用%ld打印  
7. 求绝对值可以用stdlib.h中的abs()函数
8. 使用readline函数，需要链接libreadline，然后导入主头文件
9. 字符串转数字可以用stdlib.h中的atoi()函数
10. sizeof()返回size_t类型的数据，可以用%zu格式化输出