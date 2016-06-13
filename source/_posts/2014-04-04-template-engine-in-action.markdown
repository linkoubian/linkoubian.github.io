---
layout: post
title: "基于模板引擎的新闻内容展示"
date: 2014-04-05 18:45:39 +0800
comments: true
categories: iOS, UI, 3rd-party-lib
---
在为杭州市城管委开发贴心城管客户端时，城管动态这个模块的新闻展示是把新闻分成标题、时间、贴图、正文等部分，分别展示内容的。
<!--more-->

这样做的局限性不言而喻，不能图文并茂随心所欲的在文章内部嵌入图片且图片数量有一定的限制，而且服务端在发布新闻时也不方便做文章格式的编辑等。

在设计西子快讯时，便尝试了基于模板引擎的新闻内容展示。其实这样做，开发工作量并不大，因为开源社区有了高质量的模板引擎。

## 思路
1. 根据新闻的ID从服务端获得News实例对象（含标题、附图、正文等信息）  
2. 将获得的新闻各属性信息填充到APP内的HTML模板文件中  
3. HTML模板可以指定CSS样式（同样存放在APP内）  
4. 经过模板引擎的处理得到最终的HTML内容  
5. 将处理后的HTML内容加载到WebView控件显示即可  

## 开发
### HTML模板
准备好一个名为template.html文件，需要动态替换的部分用模板标签标识。

### 显示样式
news.css也不给出了，找个前端设计师结合项目需要提供一个即可。手动替换掉template.html中的参数，加上自己的news.css，便可在桌面浏览器里测试下效果。一切OKay的话，请继续阅读本文。

### 替换模板参数
我选择的模板引擎是MGTemplateEngine，到Github下载源代码加入到Xcode工程里，然后编写如下几行代码即可。
```objc
MGTemplateEngine *engine = [MGTemplateEngine templateEngine];
[engine setMatcher:[ICUTemplateMatcher matcherWithTemplateEngine:engine]];
NSString *templatePath = [[NSBundle mainBundle] pathForResource:@"template" ofType:@"html"];
NSString *newsContentHTML = [engine processTemplateInFileAtPath:templatePath withVariables:@{@"title": @"Template based News client", @"editor": @"Linkou Bian"}];
[self.newsContentWebView loadHTMLString:newsContentHTML baseURL:[[NSBundle mainBundle] resourceURL]];
```

processTemplateInFileAtPath:withVariables中得variables参数是个字典，key对应template.html中的参数。注意示例代码中并没有把variables参数写完整。

另外html中的body_content参数对应的字符串应带有HTML标签，这样处理后的html内容放到WebView中显示效果会相对好很多。西子快讯便是这样设计与实现的。

除了在新闻顶部固定的位置显示新闻配图外，我们也可以在body_content参数对应的字符串中提供img标签以在文章任意位置插入图片。

### 操作新闻内的多媒体文件
如果需要点击图片以全屏展示，实现起来也非常简单。首先在template.html中加入一段JavaScript，
```javascript
    <script type="text/javascript">
        function extend_image(element){
            var imgTag = element.getElementsByTagName("IMG")[0];
            var url=imgTag.getAttribute("src");
            location.href="image://?url="+url;
        }
    </script>
```
然后在img标签外增加点击调用extend_image(this)相关的HTML代码即可。

用户点击图片后，WebView的委托方法webView:shouldStartLoadWithRequest:navigationType:会被调用。我们可以解析reqeust的URL字符串来做相应的处理。
```objc
if ([url hasPrefix:@"image://"]) {
    NSString *imageURL = [url substringFromIndex:[@"image://?url=" length]];
    // Present a new view controller and display the image full sreen
    // ...
    return NO;
}
```

虽然西子快讯尚未支持视频播放，但我觉得也可以采用类似的方案处理。

## 后记
个人认为采用这种方式的灵活性主要体现在，  
1. 可以定义多套显示风格（如白天、黑夜），然后将样式名传给模板  
2. 可以预设多套字体、字号（medium、large等），然后将样式名传给模板  
3. 基于自定义URL及WebView的delegate方法，实现HTML与ObjC的交互

另外，除了MGTemplateEngine外，还有其他的模板引擎可以使用，比如GRMustache。至于孰优孰劣，什么场合选择哪个，截止目前我并没有做任何研究，等接到下一个需要模板引擎的项目再说吧。