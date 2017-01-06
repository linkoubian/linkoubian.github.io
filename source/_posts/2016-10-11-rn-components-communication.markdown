---
layout: post
title: React Native 组件间数据传递
date: 2016-10-11 11:14:48 +0800
comments: true
categories: ReactNative 
---
在 React Native 项目试点过程中，封装的组件之间难免需要传递数据。本文总结了子组件如何将数据传递到使用方，以及对开发过程中遇到的一个 setState 问题的探究。

<!--more-->

## 设计一个可定制的组件
当设计一个组件供团队使用，通常使用时需要向其传递一些信息。这个过程可以参考国庆节前写的[创建一个 React Native 分隔线组件](http://balloonsys.com/blog/2016/09/30/create-your-first-reusable-react-native-component/)。

主要思路就是定义属性（propTypes），并提供默认属性值（getDefaultProps）。这里不再赘述。

## 组件返回数据给使用方
### 应用场景
假设我们的某个页面 P 使用到封装房产信息的 houseInfo 等组件，需要在 houseInfo 填写完房源总价 totalPrice 后能根据当前整个页面（含多个组件）的输入项去验证是否可以 enable 提交按钮，而且在点击提交时需要能访问到 houseInfo 等组件录入的数据（包括 totalPrice 等）。

### 组件设计
houseInfo 组件包含若干由 React Native 提供的 TextInput 组件，在 TextInput 的 onChangeText 事件绑定我们的方法：

``` js
onChangeText={this._updateTotalPrice}
```

使用到的 _updateTotalPrice 是定义在该组件里的事件响应方法，使用最精简的写法设置状态机变量：

``` js
  _updateTotalPrice(totalPrice) {
    this.setState({totalPrice}, this._onStateUpdated);
  },
```

特别需要注意的是，上述 setState 方法是异步执行的函数，将变化通知到组件的使用者的方法 _onStateUpdated 需要放在 setState 的第二个参数处，否则取 this.state.totalPrice 仍将是旧值。官方的解释如下：

>setState() does not immediately mutate this.state but creates a pending state transition. Accessing this.state after calling this method can potentially return the existing value.

在实现 _onStateUpdated 方法之前，我们先声明一个函数类型的属性：

``` js
  propTypes: {
    onGetHouseInfo: React.PropTypes.func
  },
```

这样使用者就可以传递一个函数给该组件，该组件在合适的时机调用此方法即可传递数据给使用方。

``` js
  _onStateUpdated() {
    this.props.onGetHouseInfo({totalPrice: this.state.totalPrice,
															 // 其他输入项 ...
                              }, this._validate());
  },
```

我们在传递输入的值时，还将本组件的验证结果传递出去，这样使用方即可综合各组件的 valid 值以控制提交按钮的状态。

``` js
  _validate() {
    if (XXX) {
      return false;
    }

    return true;
  },
```

### 使用方
我们在使用 houseInfo 等组件 render 表单页面 P，参考代码如下：

``` js
<HouseInfo onGetHouseInfo={this._getHouseInfo} />
```

此处的 _getHouseInfo 用于从组件接收数据，以存放在页面的 state 里面：

``` js
  _getHouseInfo(houseInfo, houseInfoValid) {
    this.setState({houseInfo, houseInfoValid});
  },
```

至于 state 的定义及初始化，放在 getInitialState 中即可，较为基础，此处略。

## 收工
👏👏👏