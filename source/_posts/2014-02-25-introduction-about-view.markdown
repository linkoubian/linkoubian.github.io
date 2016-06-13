---
layout: post
title: "关于View的用法简介"
date: 2014-02-25 15:50:00 +0800
comments: true
categories: iOS
---

本文主要针对iOS的视图相关的用法作一些介绍。
<!--more-->
## The Window
当应用启动时，UIApplicationMain会创建一个appDelegate实例。App Delegate会“持有”一个window实例（strong型）。所以window对象的生命周期与应用一致。

获得window实例有三种方式：  
1. view.window  
2. [UIApplication sharedApplication].delegate.window   
3. [UIApplication sharedApplication].keyWindow  

需要注意的是keyWindow有可能发生变化，比如当显示UIAlertView时，keyWindow为该弹出框！

## Subview and Superview
关于父子视图间的常见操作主要有：  
1. 限制子视图只在父视图区域内绘制，可设置父视图的clipsToBounds属性  
2. 判断一个视图是不是另一个的子孙视图，可以用isDescendantOfView:  
3. 子视图从父视图中移除掉，会被“释放”。若后续需使用，可另行“持有”  
4. 与视图结构相关的几个方法：  
insertSubview:atIndex:  
insertSubview:belowSubview:  
exchangeSubviewAtIndex:withSubviewAtIndex:  
bringSubviewToFront:  
5. 将父视图的所有子视图移除掉，可以用下面这种写法：  
``` objc
[view.subviews makeObjectPerformSelector:@selector(removeFromSuperView)];
```

## Visibility and Opacity
Alpha，可理解为相对于父视图的不透明度。所以即使一个视图的alpha值为1，它也有可能是透明的。

Opaque，不是用来影响视图的显示，而是用于绘图系统的一个hint值，以便绘制更加高效。

## Bounds, Center and Frame
Bounds代表的是视图自身的坐标系，而frame表示的是该视图在父视图中的位置。

改变bounds的宽高，会影响其frame，保持不变的是center值。改变bounds的origin值，相当于说该视图坐标系左上角那个点的值被改变，因此其坐标系的原点移到其他位置了。而子视图是在父视图的坐标系里定位的，所以子视图会因父视图的origin变化而逆向变化。

Center是视图bounds的中心点在父视图中的位置。Center加bounds相当于frame。

假如v2是v1的子视图，要想把v2放在v1的中心，可以这么做，
``` objc
v2.center = [v1 convertPoint:v1.center fromView:v1.superview];
```

## Transform
将视图v顺时针旋转45°，可以设置其transform属性为CGAffineTransformMakeRotation(45*M_PI/180.0)。此时v的frame是旋转后的v的外接矩形。其子视图虽然也跟着旋转，但子视图的frame并无变化，因为v的bounds不变（即v自身的坐标系不变）。

拉伸函数CGAffineTransformMakeScale会使得视图的frame发生变化，但其bounds保持不变。子视图的frame、bounds、center值均保持不变。

组合若干变换应用到视图上，下面代码先将v旋转，然后在沿着自身坐标系的x轴方向平移100个点。
``` objc
v.transform = CGAffineTransformMakeRotation(45 * M_PI/180.0);
v.transform = CGAffineTransformTranslate(v2.transform, 100, 0);
```

也可以使用CGAffineTransformConcat组合多个变换，
``` objc
CGAffineTransform r = CGAffineTransformMakeRotation(45 * M_PI/180.0);
CGAffineTransform t = CGAffineTransformMakeTranslation(100, 0);
v.transform = CGAffineTransformConcat(t,r); // not r,t
```

撤销组合变换中的某个变换，只需再次叠加其反变换。比如需要将v在新位置逆时针旋转45°，只需执行下面这行代码，
``` objc
v.transform = CGAffineTransformConcat(CGAffineTransformInvert(r), v.transform);
```

## Layout
布局除了手动设置布局以及Autoresizing之外，还有自iOS 6引入的Auto-layout。

Autoresizing发生在layoutSubviews执行之前，所以手动设置布局可放在layoutSubviews方法中。自定义的view，如果需要支持Auto-layout，需要在requiresConstraintBasedLayout返回YES。

关于Auto-layout的内容比较多，另行总结！














