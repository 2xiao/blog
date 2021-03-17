---
title: "如何用小程序实现类原生APP下一条无限刷体验"
date: 2019-06-25T11:00:19+08:00
tags: [JavaScript]
draft: false
---
这篇文章详细记录了下一条无限刷效果的实现原理，以及细节和体验优化，并将相关代码抽象成一个微信小程序代码片段，有需要的同学可查看demo源码。
<!--more-->

[原文链接](https://developers.weixin.qq.com/community/develop/article/doc/0000645ae8cf882129c8b471951c13)

## 1.背景
如今信息流业务是各大互联网公司争先抢占的一个大面包，为了提高用户的后续消费，产品想出了各种各样的方法，例如在微视中，用户可以无限上拉出下一条视频；在知乎中，也可以无限上拉出下一条回答。这样的操作方式用户体验更好，后续消费也更多。最近几年的时间，微信小程序已经从一颗小小的萌芽成长为参天大树，形成了较大规模的生态，小程序也拥有了一个很大的流量入口。

## 2.demo体验
那如何才能在小程序中实现类原生APP效果的下一条无限刷体验？

这篇文章详细记录了下一条无限刷效果的实现原理，以及细节和体验优化，并将相关代码抽象成一个微信小程序代码片段，有需要的同学可查看demo源码。

线上效果请用微信扫码体验：

![](https://user-gold-cdn.xitu.io/2019/6/27/16b96f6dfc9a5686?w=474&h=482&f=png&s=104757)

小程序demo体验请点击：[https://developers.weixin.qq.com/s/vIfPUomP7f9a](https://developers.weixin.qq.com/s/vIfPUomP7f9a)

## 3.实现原理
出于性能和兼容性考虑，我们尽量采用小程序官方提供的原生组件来实现下一条无限刷效果。我们发现，可以将无限上拉下一篇的文章看作一个竖向滚动的轮播图，又由于每一篇文章的内容长度高于一屏幕高度，所以需要实现文章内部可滚动，以及文章之间可以上拉和下拉切换的功能。

在多次尝试后，我们最终采用了在`<swiper>`组件内部嵌套一个`<scroll-view>`组件的方式实现，利用`<swiper>`组件来实现文章之间上拉和下拉切换的功能，利用`<scroll-view>`来实现一篇文章内部可上下滚动的功能。

所以页面的dom结构如下所示：

```
<swiper
  class='scroll-swiper'
  circular="{{false}}"
  vertical="{{true}}"
  bindchange="bindChange"
  skip-hidden-item-layout="{{true}}"
  duration="{{500}}"
  easing-function="easeInCubic"
>
  <block wx:for="{{articleData}}">
    <swiper-item>
      <scroll-view
        scroll-top="0"
        scroll-with-animation="{{false}}"
        scroll-y
      >
        content
      </scroll-view>
    </swiper-item>
  </block>
</swiper>

```

## 4.性能优化
我们知道view部分是运行在webview上的，所以前端领域的大多数优化方式都有用。例如减少代码包体积，使用分包，渲染性能优化等。下面主要讲一下渲染性能优化。

### 4.1 dom优化
由于页面需要无限上拉刷新，所以要在`<swiper>`组件中不断的增加`<swiper-item>`，这样必然会导致页面的dom节点成倍数的增加，最后非常卡顿。

为了优化页面的dom节点，我们利用`<swiper>`的`current`和`<swiper-item>`的`index`来做优化，控制是否渲染dom节点。首先，仅当`index <= current + 1`时渲染`<swiper-item>`，也就是页面中最多预先加载出下一条，而不是将接口返回的所有后续数据都渲染出来；其次，对于用户已经消费过的之前的`<swiper-item>`，不能直接销毁dom节点，否则会导致`<swiper>`的`current`值出现错乱，但是我们可以控制是否渲染`<swiper-item>`内部的子节点，我们设置了仅当`current <= index + 1 && index -1 <= current`时才会渲染`<swiper-item>`中的内容，也就是仅渲染当先文章，及上一篇和下一篇的文章内容，其他文章的dom节点都被销毁了。

这样，无论用户上拉刷新了多少次，页面中最多只会渲染3篇文章的内容，避免了因为上拉次数太多导致的页面卡顿。

### 4.2 分页时setData的优化

#### setData工作原理

![](https://user-gold-cdn.xitu.io/2019/6/27/16b96f71c1e55aa6?w=1960&h=1452&f=jpeg&s=378305)

小程序的视图层目前使用`WebView`作为渲染载体，而逻辑层是由独立的 `JavascriptCore` 作为运行环境。在架构上，`WebView` 和 `JavascriptCore` 都是独立的模块，并不具备数据直接共享的通道。当前，视图层和逻辑层的数据传输，实际上通过两边提供的 `evaluateJavascript` 所实现。即用户传输的数据，需要将其转换为字符串形式传递，同时把转换后的数据内容拼接成一份 `JS` 脚本，再通过执行 `JS` 脚本的形式传递到两边独立环境。

而 `evaluateJavascript` 的执行会受很多方面的影响，数据到达视图层并不是实时的。

* 每次 `setData` 的调用都是一次进程间通信过程，通信开销与 setData 的数据量正相关。
* `setData` 会引发视图层页面内容的更新，这一耗时操作一定时间中会阻塞用户交互。
* `setData` 是小程序开发中使用最频繁的接口，也是最容易引发性能问题的接口。

#### 避免不当使用setData

* `data` 应仅包括与页面渲染相关的数据，其他数据可绑定在this上。使用 `data` 在方法间共享数据，会增加 setData 传输的数据量，。
* 使用 `setData` 传输大量数据，通讯耗时与数据正相关，页面更新延迟可能造成页面更新开销增加。仅传输页面中发生变化的数据，使用 `setData` 的特殊 `key` 实现局部更新。
* 避免不必要的 `setData`，避免短时间内频繁调用 `setData`，对连续的setData调用进行合并。不然会导致操作卡顿，交互延迟，阻塞通信，页面渲染延迟。
* 避免在后台页面进行 `setData`，这样会抢占前台页面的渲染资源。可将页面切入后台后的`setData`调用延迟到页面重新展示时执行。

#### 优化示例
无限上拉刷新的数据会采用分页接口的形式，分多次请求回来。在使用分页接口拉取到下一刷的数据后，我们需要调用`setData`将数据写进`data`的`articleData`中，这个`articleData`是一个数组，里面存放着所有的文章数据，数据量十分庞大，如果直接`setData`会增加通讯耗时和页面更新开销，导致操作卡顿，交互延迟。

为了避免这个问题，我们将`articleData`改进为一个二维数组，每一次`setData`通过分页的 `cachedCount `标识来实现局部更新，具体代码如下：

```
this.setData({
  [`articleData[${cachedCount}]`]: [...data],
  cachedCount: cachedCount + 1,
})
```
`articleData`的结构如下：

![](https://user-gold-cdn.xitu.io/2019/6/27/16b96f75325b4cd3?w=1096&h=1384&f=png&s=273961)

### 4.3 体验优化
解决了操作卡顿，交互延迟等问题，我们还需要对动画和交互的体验进行优化，以达到类原生APP效果的体验。

在文章间上拉切换时，我们使用了`<swiper>`组件自带的动画效果，并通过设置`duration`和`easing-function`来优化滚动细节和动画。

当用户阅读文章到底部时，会提示下一篇文章的标题等信息，而在页面上拉时，由于下一篇文章的内容已经加载出来了，这样在滑动过程中会出现两个重复的标题。为了避免这种情况出现，我们通过一个占满屏幕宽高的空白`<view>`来将下一篇文章的内容撑出屏幕，并在滚动结束时，通过`hidden="{{index !== current && index !== current + 1}}"`来隐藏这个空白`<view>`，并对这个空白`<view>`的高度变化增加动画，来实现下一篇文章从屏幕底部滚动到屏幕顶部的效果：

```
.fake-scroll {
  height: 100%;
  width: 100%;
  transition: height 0.3s cubic-bezier(0.167,0.167,0.4,1);
}
```


![](https://user-gold-cdn.xitu.io/2019/6/27/16b96f786e3320d0?w=750&h=1442&f=png&s=67753)

而当用户想要上拉查看之前阅读过的文章时，我们需要给用户一个“下滑查看上一条”提示，所以也可以采用同上的方式，通过一个占满屏幕宽高的提示语`<view>`来将上一篇文章的内容撑出屏幕，并在滚动结束时，通过`wx:if="{{index + 1 === current}}"`来隐藏这个提示语`<view>`，并对这个提示语`<view>`的透明度变化增加动画，来实现下拉时提示“下滑查看上一条”的效果：

```
.fake-previous {
  height: 100%;
  width: 100%;
  opacity: 0;
  transition: opacity 1s ease-in;
}
.fake-previous.show-fake-previous {
  opacity: 1;
}
```

至此，这个类原生APP效果的下一条无限刷体验的需求的所有要点和细节都已实现。

记录在此，欢迎交流和讨论。

小程序demo体验请点击：[https://developers.weixin.qq.com/s/vIfPUomP7f9a](https://developers.weixin.qq.com/s/vIfPUomP7f9a)

