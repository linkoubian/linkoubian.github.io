---
layout: post
title: 创建一个 React Native 分隔线组件
date: 2016-09-30 10:24:06 +0800
comments: true
categories: ReactNative
---
在移动端表单页面里有不少分隔线绘制需求，构造一个组件以在不同页面复用是个很自然的做法。

<!--more-->

## 目标
期望在各页面导入 require 该组件后，直接使用如下的方式绘制分隔线：  

``` js
<Line height={1} paddingLeft={10} paddingRight={0} backgroundColor='white' />
```

当然也为上述四个属性提供默认值，如果默认值符合需求，则只需：  

``` js
<Line />
```

## 所需技术
### 创建类
我采用的是 ES 5 的语法创建类的，代码如下：  

``` js
let Line = React.createClass({
  render() {
    return (
    );
  },
});

module.exports = Line;
```

其中 render 方法类似于 Native 开发中视图的绘制方法。我们稍后在其 return 中加上构建与 React Native 基础组件之上的 XML 代码。

### 添加属性
构建该分隔线组件给客户程序使用，需要让别人知道支持哪些属性，比如线的高度、左右缩进、背景色及线的颜色等。

``` js
  propTypes: {
    height: React.PropTypes.number,
    paddingLeft: React.PropTypes.number,
    paddingRight: React.PropTypes.number,
    backgroundColor: React.PropTypes.string,
    lineColor: React.PropTypes.string,
  },
```

为了用起来方便，我们还可以给这些属性提供默认值：

``` js
  getDefaultProps() {
    return {
      height: 0.5,
      paddingLeft: 0,
      paddingRight: 0,
      backgroundColor: '#FFFFFF',
      lineColor: '#D3D3D3',
    }
  },
```

### 绘制控件
具体的绘制代码在 render 方法中完成。如前面所述，我们将用到 React Native 提供的基础组件，所以我们需要在代码顶部导入。

``` js
import React, { Component } from 'react';
import { View } from 'react-native';
```

此时我们就可以在 render 方法的 return 语句中加上下面代码以完成分隔线的绘制了：

``` xml
      <View style={} >
        <View style={} />
      </View>
```

外层的 View 的 style 属性设置为：  

``` js
{backgroundColor: this.props.backgroundColor, height: this.props.height, paddingLeft: this.props.paddingLeft, paddingRight: this.props.paddingRight}
```

内层的 View 的 style 属性设置为：

``` js
{flex: 1, backgroundColor: this.props.lineColor}
```

## 总结
本文选取最最简单的分隔线介绍 React Native 实践过程中的代码编写思路，完成的其他控件有表单填写进度等。

