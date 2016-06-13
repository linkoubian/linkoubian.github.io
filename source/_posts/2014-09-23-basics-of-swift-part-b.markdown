---
layout: post
title: Swift 语言基础（下）
date: 2014-09-23 10:11:08 +0800
comments: true
categories: Swift
---
本文主要记录Swift中关于Optional Type、集合等知识点。

<!--more-->

## 可选类型（Optional Type）

处于安全的设计考量，在Swift中未初始化的String不可以println。声明时设为nil后，println也不行，因为nil不是STR。

所以引入Optional Type，可以理解为对另一个类型的装箱（由本人引自Java的术语）。String?便是一个Optional Type，它是对String的包装。声明时不需要赋值为nil：var str: String?。此时println打印出nil。如果赋值为“Hello World”，打印Optional(“Hello World”)。

### 手动拆箱（Manual Unwrapping）

若需要拆箱，可以if let unwrappedStr = str {}。如果str为nil，if分支的代码块不会执行。值得注意的是，1）想要获得Optional Type所封装的那个值，必须用if语句拆箱；2）unwrappedStr的作用域为if代码块。

### 强制拆箱（Forced Unwrapping）

如果明确的知道Optional Type包含一个值，则可以使用强制拆箱（Forced Unwrapping）。具体写法是这样的：println(str!)。不必经过前面提到的if就可以拿到被包含的值。

### 隐式拆箱（Implicit Unwrapping）

强制拆包是声明Optional Type时用？，访问被包含的值时加个！。而隐式拆箱与之不同的是，声明时用！，访问被包含的值时什么都不需要加，就好像没有使用过Optional Type似的。

### 可选链（Optional Chaining）

声明一个Optional Type：var str: String?
可选消息链：let upperStr = str?.uppercaseString

1）可选链可以级联多次，这样便客观的降低if嵌逃层级   
2）使用Optional Chaining代码更简洁

## 集合

1) 无论是数组还是字典，都是用[]   
注：ObjC里字典用的是{}   
2）默认情况下，数组只能放同一类型的元素，除非显示或隐式声明为[AnyObject]类型   
3）将字典某key对应的值设为nil，即为删除   
4）字典的value为Optional Type   
5）数组、字典，都是拷贝赋值   
6）使用let声明的常量集合，不可以append、remove   
