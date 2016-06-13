---
layout: post
title: "基于iCarousel构建新闻头条"
date: 2014-03-24 16:25:24 +0800
comments: true
categories: iOS, UI, 3rd-party-lib
---
市面上大部分新闻客户端的布局，都是顶部放一个可以左右滑动的头条，下方是新闻列表。像网易新闻的头条部分，通常是新闻与广告穿插。因为近期正在开发类似的一个APP，现总结头条部分的构建步骤。
<!--more-->
## 技术选型
去code4app.com上搜索ScrollView相关的代码，会找到一些类似的代码示例，比如这个[EScrollView](http://code4app.com/ios/EScrollerView/51935e166803fac572000003)。如果是本着学习的出发点，借鉴其思路并酌情修改其代码以满足自身项目需求，也不是难事。但如果要用在商业项目里，并期望获得足够的灵活性可能还需花费很多精力。

经过一番Google，在Github里找到了[iCarousel](https://github.com/nicklockwood/iCarousel)这个控件，其示例代码中含不少例子，不过大部分例子的效果都是CoverFlow风格。但其中的Paging Example给了我灵感，把iCarousel View的高度设小一点，再额外加个page control及text label不就刚好么？欣喜之余，快速打开该示例的代码仔细阅读。

下面详细介绍如何基于iCarousel构建新闻头条。

## iCarousel简介
### 定制iCarousel视图
拖一个UIView到View Controller的顶部，大小为320X190，类名设置为iCarousel，连接到一个名为headlinesView的Outlet。

在viewDidLoad中调用如下方法，
``` objc
- (void)setupHeadlinesView
{
    self.headlinesView.type = iCarouselTypeLinear;
    self.headlinesView.pagingEnabled = YES;
    self.headlinesView.dataSource = self;
    self.headlinesView.delegate = self;
}
```
此处关键的一个属性便是pagingEnabled，设置后刚好符合期望的效果。

### 为iCarousel提供数据
iCarousel采用类似于table view的设计方式，提供了不少datasource及delegate方法。

下面这个方法告诉iCarousel共有几个cell item，
``` objc
- (NSUInteger)numberOfItemsInCarousel:(iCarousel *)carousel
{
    return kCoundOfHeadlineNews;
}
```

当需要加载某个cell view时，会调用viewForItemAtIndex，我们需在该方法里确保用于显示新闻头条图片的imageView被构建并设置相应地image。
``` objc
- (UIView *)carousel:(iCarousel *)carousel viewForItemAtIndex:(NSUInteger)index reusingView:(UIView *)view
{
    UIImageView *headlineImageView = nil;
    
    // Create new view if no view is available for recycling
    if (view == nil) {
        view = [[UIView alloc] initWithFrame:self.headlinesView.bounds];
        headlineImageView = [[UIImageView alloc] initWithFrame:view.bounds];
        [headlineImageView setImage:[UIImage imageNamed:@"Headline_Placehold"]];
        headlineImageView.tag = kHeadlineImageViewTag;
        [view addSubview:headlineImageView];
    }
    else {
        headlineImageView = (UIImageView *)[view viewWithTag:kHeadlineImageViewTag];
    }

    // Load image for headline news
    if (index < self.headlineNews.count) {
        [headlineImageView setImageWithURL:[NSURL URLWithString:[self.headlineNews[index] img]]];
    }
    
    return view;
}
```
上面代码中的headlineNews是一个数组，用来存从远程服务器获得的头条新闻（Model）列表，其中img属性（头条新闻的字段）为对应图片的URL。当加载cell view时headlineNews可能还没有数据，所以上述代码做了个判断，即if (index < self.headlineNews.count)。

当远端数据加载完毕，我们可以调用[self.headlinesView reloadData]以更新界面。

另外，往Image View设置远程图片，可以用SDWebImage提供的方法setImageWithURL，该框架可以有效的管理图片的缓存。

### 定制iCarousel的行为
当头条新闻滑到最后一页时，通常需要能够回到第一页。iCarousel提供了一个delegate方法以定制该控件的属性。
``` objc
- (CGFloat)carousel:(iCarousel *)carousel valueForOption:(iCarouselOption)option withDefault:(CGFloat)value
{
    switch (option) {
        case iCarouselOptionWrap:
            return YES;
        default:
            return value;
    }
}
```

点击某一个cell view时，需要展示新闻详情页面。跳转的逻辑可以放在carousel:didSelectItemAtIndex:中。

## 添加新闻标题
标题（Label）与当前页指示控件（page control）并不属于iCarousel的cell view，因为整个滑动过程中只有一个Label和一个Page Control。所以我们可以在Storyboard中拖放相应控件作为iCarousel的子视图，再设置属性即可。

当滑动后，到达新页面时，可以在carouselCurrentItemIndexDidChange:中设置Label及Current Page。
