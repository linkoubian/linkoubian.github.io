---
layout: post
title: "hitTest示例"
date: 2014-04-08 14:53:20 +0800
comments: true
categories: iOS, UI
---
早上翻看Github本周排名靠前的Objective-C开源项目，瞄了几眼[PWParallaxScrollView](https://github.com/wpsteak/PWParallaxScrollView)的代码。发现其在ScrollView中嵌入Button，然后为了使得Button能接受点击事件是借助- hitTest:withEvent:实现的。
<!--more-->
一番Google，大致知道hitTest是干嘛的了。顺道写个Sample，改变默认的消息响应流。

## 准备
建立一个Single VC的工程，添加两个有部分重叠的Button，分别关联上touchUpInSide处理方法。

```objc
- (IBAction)buttonAPressed:(id)sender
{
    NSLog(@"You pressed button A");
}

- (IBAction)buttonBPressed:(id)sender
{
    NSLog(@"You pressed button B");
}
```

因为Button B(tag为102)是叠在Button A(tag为101)上面的，所以在重叠区域点击，控制台输出的是You pressed button B。这种情况下，如果希望收到消息的是Button A呢？那就用hitTest吧！

## 实现
建立一个UIView的子类，然后重写- hitTest:withEvent:方法，
```objc
- (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event
{
    for (UIView* subview in self.subviews) {
        CGPoint convertedPoint = [self convertPoint:point toView:subview];
        UIView *result = [subview hitTest:convertedPoint withEvent:event];
        
        if (result.tag == 101){
            return result;
            break;
        }
    }
    
    return [super hitTest:point withEvent:event];
}
```

紧接着，把Button的父视图，即根视图view对应的类设为刚创建的类型。运行，测试，看到效果了吧?!

## 后记
源代码里有乾坤，不注意看就错过了:-)