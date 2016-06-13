---
layout: post
title: Swift 语言基础（上）
date: 2014-09-21 22:59:56 +0800
comments: true
categories: Swift
---
周五花了54刀买了RW网站上的Swift by Tutorials，共261页。平均一页1块多软妹币啊！！！朋友圈有评论说：“很庆幸，只需要看两三百页，得其精髓，最怕书长！”。深以为然！

<!--more-->

本系列文章，为Swift by Tutorials的读书笔记。

## 从Hello World开始
为了耍耍Swift，首先得打开Xcode 6。在弹出的工程模板里选哪一个呢？这是个问题！

暂时的答案是，一个都不选，关掉！然后在File/New菜单中选择Playground即可。

啥都不说了，在打开的playground编辑区，直接输入println(“Hello World”)吧，此时便可以在右侧看到即时效果。

如果想看程序的运行结果，可以在View/Assistant中选择Show Assistant Editor。

## 变量&常量
声明一个变量的方式：var str: String = “Hello World”，其中类型可以省略。

需要注意的是，类型可以省略不写，不代表Swift是动态类型语言。比如尝试给str变量赋23这样的整数值，就会报错。其类型由首次赋值推断出，此过程为Type Inference。

前面我们解释了类型可以省略不写，现在我们谈谈此处省略的String。此String相当于Obj-C的NSMutableString，是可变的。所以你可以进行这样的操作，str = str + “, I love you!”。

那如何创建一个“不可变”的字符串呢？将var换成let即可。使用let后，我们既不可以str = “Hello Swift”，也不可以str.append。

## 数值类型
浮点数与整数运算，需要显式的转换类型。Int.max + 1会直接报错而不是变成一个负数。

## 逻辑类型
var knowsSwift: Bool = true。此处只可以赋值true或者false，不像其他语言可以使用0或非0值。

## 元组
var product = (“iPhone 6”, 6000)。要输出商品名，我们可以使用println(product.0)。如果加价，可以这么写：product.1 += 100。

如果将元组product拆解成单个元素以方便访问，可以这么做：let (name, price) = product。这时，我们除了使用product.0访问商品名，还可以直接用name。

我们在声明并赋值元组时，还可以指定“属性名”：var product = (name: “iPhone 6”, price: 6000)。

## 字符串操作
输出product的信息，我们可以这么做：println("Price of \(product.name) is \(product.price)")。此为Swift的String Interpolation特性。相对于NSLog的format参数，方便不少。

另外，我们在使用String Interpolation时，还可以直接操作相关的变量。比如println("Price of \(product.name) is \(product.price + 100)”)。

## 控制流
### for循环
for i in 1…5 {println (“\(i)”)}
其中…为闭区间操作符（Closed Range Operator），相当于Range(start: 1, end: 6)。

### while循环
```sh
var i = 0
while i < 5 {
	println(“\(i)”)
	i++
}
```

### if语句
即使待执行的只有一行代码，也需要{}。

### switch语句
Swift中，switch可以使用任何类型，不需要break语句，如果case不完备则必须有default语句。

case可以这么写：case “up”, “down”: println(“you pressed up or down”)。case还可以这么写：case 1..<10: println(“between 1 and 10, 10 not included”)。

— EOF —