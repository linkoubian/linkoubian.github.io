---
layout: post
title: React Native 项目实战
date: 2016-10-18 15:02:29 +0800
comments: true
categories: ReactNative 
---
本文以 Twitter 工程师 Bonnie Eisenman 撰写的 Learning React Native 一书的第九章内容为蓝本，总结了 React Native 项目实践的一些经验。

<!--more-->

## 项目组织结构
所有的新增文件均放在项目根目录下的 src 里，主要有包含各页面的 components 子目录、数据模型的 data 子目录、负责数据存取的 stores 子目录、公共样式定义 styles 子目录、存放资源图片等的 resources 子目录等。

![RN Project Structure](https://c2.staticflickr.com/6/5781/30365572286_b0c35d2176_o.png =226x)

index.ios.js 是 Native 应用的入口，该文件尽量简洁，指向 RN 项目里负责页面跳转的主文件 components/Zebreto.js。

## 案例项目简介
作者提供的示例项目包含三个页面，包含多副牌（Deck）的列表页、为选中的某副牌增加一张卡牌（Card）的页面、为某张卡牌选择答案（Review）的页面。

在列表页点击 Create Deck 按钮，按钮上方出现输入框，填写内容作为 Deck 的名称。点击 Deck 右侧按钮则进入增加卡牌页面，点击 Deck 左侧则进入 Review 的页面。

## 构建基础组件
### 公共样式
全局的颜色、字号等，可以在 js 里直接定义字典数据结构，然后 exports 以供外界访问。比如：

``` js
var palette = {
  pink: '#FDA6CD'
};

module.exports = palette;
```

所以，像平安好房的应用可以参照上述结构定义 pa_orange 等色值。值得注意的是，我们也可以在一个 js 文件里定义多个字典，然后 exports 时将他们包含在花括号中即可。

``` js
module.exports = {fonts, color};
```

在使用上述结构时，就需要这么 import 了：

``` js
import { fonts, color } from '<PATH-OF-FILE>';
```

### 公共组件
我们不直接用 Text 组件，而是包装成 HeadingText 和 NormalText 供项目里的不同页面使用。同样的，为了代码重用与使用便捷，我们封装 Button、Input、LabeldInput 等组件。

其中 Button 组件构建在 TouchableOpacity 基础之上，支持 func 类型的属性以在点击时调用使用方的方法，也支持 View.propTypes.style 类型的属性以方便定制其样式等。

需要注意的是，为了让 Button 组件能包含其他子组件，我们使用了一个 object 类型的属性，然后在 render 时输出 children 即可。

HeadingText 和 NormalText 建立在 Text 组件之上，自定义样式是通过 Text.propTypes.style 类型的属性支持的。注意此处类型不同于前面 Button 使用过的样式类型。

Input 组件建立在 TextInput 之上，LabeledInput 组合了 Input 和 NormalText 两个组件，体现了复用的理念。

## 页面开发
### Deck 列表页
#### 数据建模
在 React Native 项目试点过程中，尚不熟悉 JavaScript 的类相关语法。当时都是用的字典:[

``` js
class Deck {
  constructor() {
  }
}

module.exports = Deck;
```

数据在本地使用 AsyncStorage 存取，所有的存取方法均封装在 DeckStore.js 里。

#### Reflux 架构
作者使用 Reflux 架构实现数据的单项流动，主要的两个概念便是 Store 和 Action 了。

用户在 View 上操作，触发 Action，示例 View 的事件响应代码如下：
``` js
DeckActions.createDeck(deck)
```

而 createDeck 是在 actions.js 里定义的：
``` js
export var DeckActions = Reflux.createActions([
  'createDeck'
]);
```

Store 的 init 方法里监听 Action：
``` js
var decksStore = Reflux.createStore({
  init() {
    this._decks = [];
    // ...
    this.listenTo(DeckActions.createDeck, this.createDeck);
    // ...
  },

	// ...

  createDeck(deck) {
    this._decks.push(deck);

    this.emit();
  }

	// ...

  emit() {
	  // ...
    this.trigger(this._decks);
  },
});

module.exports = decksStore;
```

这样在 createDeck 这一 action 触发后，Store 变回执行 createDeck 方法以更新 Store 中的数据，再通过 emit 方法通知出去。

View 里面监听 DeckStore 的消息，将通知携带来的数据模型存在 state 里以触发 render 方法的执行（更新 UI）。

![Reflux Architecture](https://c2.staticflickr.com/6/5816/29772102074_b8ec9b3328_o.png =328x)

#### 页面组装
src/components/Decks/index.js 是该页面的主文件，会包含一些子组件以完成整个页面的渲染。

在 index.js 的 render 方法里，将这一段代码封装在一个独立的工具方法，然后在 render 里面引用其返回的 UI 结构。

``` js
  _getDecks() {
    if (!this.state.decks) {
      return null;
    }

    return this.state.decks.map((deck) => {
      return (
        <Deck deck={deck} .../>
      );
    });
  },
```

使用高阶函数 map 根据每个 deck 数据对象生成对应的 Deck 标签，作为数据返回。

``` js
  render() {
    return (
      <View>
        {this._getDecks()}
        <DeckCreation .../>
      </View>
    );
  }
```

输入 Deck 名时比正常情况下多一行输入框，所以在 DeckCreation.js 中根据一个 state 变量分别返回不同的两个子组件：EnterDeck 和 CreateDeckButton。前者由公共组件 Input 和 CreateDeckButton 组合而成。CreateDeckButton 由公共组件 Button 和 NormalText 组合而成。

详细代码不在本文中提供，以思路为重。

### Card 新建页
整体过程类似于 Decks 页面的构建。主要就是在 View 触发 CardActions.createCard 这一 Action，在 Card 的 Store 中监听以更新数据集合。

### Review 交互页
本页面有两种场景，若存在尚未 Review 过的 Card 则显示可选择答案的 Review 页面，否则显示 Review 结果（正确率）。所以 src/components/Review/index.js 里分两种情况返回 UI 结构。

选择答案的 UI 结构，其封装在 ViewCard.js 中，做法类似于之前 Decks 利用 map 高阶函数的方案。Review 这一块稍微难懂一点的是其 Store 里根据录入的卡片构造选项的逻辑，但这其实已不是 React Native 的范围，耐心的多看一会儿就可以懂。

## 问题与解决方案
### Decks 页面不展示模拟数据
我在完成 Decks 页面的展示时，就不等 Create Deck 功能的实现，就开始测试一下页面。比如在 components/Decks/index.js 的 getInitialState 中直接构造几个 Deck 对象。但是并没有展示出来:[

原因在于 Store 发出的消息，导致 View 的 state 里的数据立即被置空。我们可以临时在加个判断，为空就不 setState({decks}) 即可。

### Review 结果展示页告警
该页使用公共组件 NormalText 并传递 color 给它，但作者提供的 NormalText 代码里使用的是 View.propTypes.style，应改为 Text.propTypes.style！


