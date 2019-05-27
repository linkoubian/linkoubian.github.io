---
layout: post
title: "Avoid Nested Push Animation"
date: 2015-07-04 21:28:39 +0800
comments: true
categories: iOS Navigation Nested-Push
---
在iOS 8里使用UINavigationController把一个页面Push出来后，很短的时间内再次Push一个页面，并没有太大问题，两个页面相继显示。但在iOS 7里运行会遇到nested push animation can result in corrupted navigation bar问题。

<!--more-->

## 模拟项目
### 方法一
给一个按钮添加点击事件，在事件响应里：
``` objc
    UIViewController *detailPage = [UIViewController new];
    UIViewController *anotherPage = [UIViewController new];

    [self.navigationController pushViewController:detailPage animated:YES];

    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, 0.3 * NSEC_PER_SEC), dispatch_get_main_queue(), ^{
        [self.navigationController pushViewController:anotherPage animated:YES];
    });
```
这样一点击此按钮，便可模拟本文着重讨论的问题。

### 方法二
实际项目里有两个基于View实现的控件，点击时首先缩放一下控件里的图标，然后push一个页面。当点击控件A后，若快速点击控件B，就会发生nested push问题。

方法二中控件带有一定时长的缩放动画，给了用户足够的反应时间去连续点击。而方法一中是基于GCD代码模拟，可精确控制nested push的间隔。示例代码参见[PushVC示例之Initial Commit](https://github.com/linkoubian/PushVC)

## 问题剖析
首先，从UINavigationController这一层面考虑，在第一个push的动画还没完成，到来的第二个push请求，是否可以忽略？印象中貌似iOS 6就是这么处理的。

然后，我们也可以从UI层面控制，尽量不要在push过程中发起第二次push请求。

## 解决方案
UINavigationController在push过程中，如何忽略其他push请求呢？查看其Delegate，发现其中有如下两个方法：
``` objc
- navigationController:willShowViewController:animated:
- navigationController:didShowViewController:animated:
```

所以，我们可以基于UINavigationController建立NavController，增加一个名为inTransition的BOOL属性。将NavController的delegate指向自身，并实现上述两个delegate方法。在willShow中将inTransition置为YES，在didShow中置为NO。接下来重写pushViewController:animated方法：
``` objc
- (void)pushViewController:(UIViewController *)viewController animated:(BOOL)animated {
    
    if (self.inTransition) {
        NSLog(@“Ignore push request when top page is still animating”);
        return;
    }
    
    [super pushViewController:viewController animated:animated];
}
```

此时运行模拟项目，快速点击两个控件后，控制台会出现“Ignore push request when top page is still animating”消息，说明使用NavController可以像iOS 6的UINavigationController那样成功避免Nested Push问题。

如前分析，我们也可以在UI层做控制，避免在Push过程中发起Push请求。仔细分析一下，第二次Push请求可能在控件的动画过程中发起，也可能在第一次Push过程中发起。所以，控件（示例中是基于View实现的）是否响应事件（示例中是基于tapGesture实现的）需要考虑两个条件。

## 前端控制
如何在动画过程中，避免另一个控件发起Push请求？我们可以在第一个控件的动画过程中，忽略触摸事件即可。核心代码是UIApplication的beginIgnoringInteractionEvents和endIgnoringInteractionEvents方法。主要代码如下：
``` objc
- (void)tapped {
    
    if (self.iconView) {
        if ([self.iconView.layer animationForKey:TAP_ANIMATION_KEY] == nil) {
            [self.iconView.layer addAnimation:[self tapAnimation] forKey:TAP_ANIMATION_KEY];

            [[UIApplication sharedApplication] beginIgnoringInteractionEvents];
            self.inAnimating = YES;
            
        } else {
            NSLog(@“Do NOT tap until animation for %@ stopped”, self);
        }
    } else {
        [super tapped];
    }
}

- (void)animationDidStop:(CAAnimation *)anim finished:(BOOL)flag {
    
    [[UIApplication sharedApplication] endIgnoringInteractionEvents];
    self.inAnimating = NO;
    
    [super tapped];
}
```

当然也可以判断是否有其他控件处于inAnimating状态，以决定是否忽略当前控件的触摸事件（gestureRecognizer:shouldReceiveTouch:）。

如何避免在Push过程中再次发送Push请求呢？我们可以在UIGestureRecognizerDelegate方法（方法名前面刚刚提过）中判断：
``` objc
- (BOOL)gestureRecognizer:(UIGestureRecognizer *)recogniser shouldReceiveTouch:(UITouch *)touch {
    
    if ([self.navigationController isKindOfClass:[NavController class]]) {
        NavController *nav = (NavController *)self.navigationController;
        return !nav.inTransition;
    }
    
    return YES;
}
```

完整的代码可以从Github获得，[PushVC之最新commit](https://github.com/balloonsys/PushVC)。

**注**：至于如何在不同commit之间切换代码，本文不再赘述。

## 参考资料
1. 苹果官方文档：[UINavigationControllerDelegate](https://developer.apple.com/library/ios/documentation/UIKit/Reference/UINavigationControllerDelegate_Protocol/) 
2. 苹果官方文档：[Turn off delivery of touch events for a period](https://developer.apple.com/library/ios/documentation/EventHandling/Conceptual/EventHandlingiPhoneOS/multitouch_background/multitouch_background.html)  
