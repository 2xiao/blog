---
title: "【小程序】炫酷点赞动画实现方案"
date: 2019-06-25T11:00:19+08:00
tags: [JavaScript]
draft: false
---
如何用CSS实现一个复杂的点赞动画
<!--more-->

## 1.效果演示
在小程序中有很多用户交互的场景，例如转发，点赞等，之前的点赞一般采用两张「未点赞 / 已点赞」的图片做成雪碧图来实现，这样用户点击时只能感受到一个静态的生硬的变化。为了加强点赞后的反馈感，我们给点赞增加了动画效果，如下：

![](http://mat1.gtimg.com/www/js/news/like4.gif)

这种效果是如何实现的呢？

## 2.实现分析
在css中，有一个计时函数`animation-timing-function`属性，该属性可以作用于整个动画中，定义了动画的每次循环是如何随时间递进的，属性的可能值为一或多个` <timing-function>`，比如有
* `linear`
* `ease`
* `ease-in`
* `ease-out`
* `ease-in-out`
* `cubic-bezier(n,n,n,n)`
* 阶跃函数`steps(n,start/end)`。

对于关键帧动画来说，`timing function`作用于一个关键帧周期而非整个动画周期，即从关键帧开始开始，到关键帧结束结束。各个关键帧都有单独的计时函数。这时的计时其实指的的帧之间的时间函数，顺序播放的动画，结尾关键帧的计时函数不会生效。

在这个动画中，我们正是使用了`animation-timing-function`的阶跃函数`steps()`，使用这个函数来控制一组包含渐变效果的点赞图（如下图），按照设置好的合适的步数和时间间隔播放，同时在水平轴跳跃的移动图片，就能达到我们的效果。

![](http://mat1.gtimg.com/www/js/news/like_icon_wx.png)

## 3.代码实现
使用上文提到的一张特殊的多帧点赞图作为背景

```
.agree-icon {
  width: 50rpx;
  height: 50rpx;
  background-repeat: no-repeat;
  background-image: url('./like_icon.png');
  background-position: left;
  background-size: 1400% 100%;
}
```

设置动画`animation-timing-function`的阶跃函数`steps()`，这里需要明确一下，`steps()`是一个阶跃函数，函数曲线不是连续的，因为图片一共有14张，存在13个间隔，所以，我们设置阶跃的步数为13。

当用户点赞时，我们为`.agree-icon`增加一个`.like-active`的属性，此属性设置了背景图片按照`@keyframes plus-one`中所设置的样式从左往右移动，移动的动画为`animation-timing-function`所设置的13步，还可以通过设置`animation-duration`的值来控制动画的总时长。

```
.like-active {
  animation-timing-function: steps(13);
  animation-name: plus-one;
  animation-duration: .58s;
  animation-iteration-count: 1;
  display: inline-block;
  animation-fill-mode: forwards;
}

@keyframes plus-one {
  0% {
    background-position: left;
  }
  100% {
    background-position: right;
  }
}
```

## 4.进阶动画
下面我们来看一个更复杂的点赞动画，用户每次点击都会有大小和方向随机的表情包四散飞射出来，同样可以使用CSS的计时函数`animation-timing-function`属性来实现。

![](http://mat1.gtimg.com/www/js/news/likes.gif)

我们可以将这个复杂的动画分解，先实现单次点击的点赞动画（如下图），实现原理和上述方法一样，也是用一张特殊的多帧点赞图作为背景，然后设置
`animation-timing-function`和`background-size`属性。

![](http://mat1.gtimg.com/www/js/news/rocket-eg.png)

接下来，只需要循环渲染出此动画元素，并且使用随机函数来控制元素有不同的大小和位置即可。

具体代码如下：

```
getAniCon: function () {
  const aniCon = []
  for (let i = 0; i < 21; i++) {
    const angle = (Math.random() - 0.6) * 160 // 旋转角度为左右各0-80度
    const scale = Math.random() * 0.8 + 1     // 缩放比例为1-180%
    const opacity = Math.random() * 0.1 + 0.9 // 透明度范围为60%-100%
    const bottom = i > 10 ? 0 : -54
    const left = (i - 1) % 10 * -264

    const item = {}
    item.angle = angle
    item.numStyle = `background-position: bottom ${bottom}rpx left ${left}rpx;`
    item.rocketStyle = `transform: rotate(${angle}deg);
      width: ${scale * 500}rpx;
      height: ${scale * 250}rpx;
      background-size: ${scale * 6000}rpx ${scale * 250}rpx;
      bottom: ${scale * 30}rpx;
      right: ${scale * -150}rpx;`
    item.opacity = opacity
    aniCon.push(item)
  }
  return aniCon
},
```

为了方便理解和区分，我为每一个点赞动画都加一个灰色背景，如下：

![](http://mat1.gtimg.com/www/js/news/like.gif)

## 5.性能优化
使用循环渲染出来的点赞动画虽然简单粗暴，但是会带来很多性能问题，比如：dom元素冗余导致的卡顿，动画元素的`z-index`过高导致的`click`事件被遮挡等，所以，我们还需要对此动画进行一些优化。

针对循环渲染出来的dom元素冗余导致的卡顿问题，我们使用两个参数来优化，一个是**是否正在点赞**，一个是**点赞次数**，其中，是否正在点赞是通过截流函数`debounce`，对点击事件增加500毫秒间隔来判断的，仅当用户正在点赞，且当前动画的`index`值满足`index >= clickTimes || index + 3 <= clickTimes`时，才会渲染该动画元素，这样页面中最多会有4个动画节点在同时播放，其他的动画节点都会被销毁。

针对动画元素的`z-index`过高导致的`click`事件被遮挡的问题，我们同样可以通过**是否正在点赞**这个参数，来动态控制点赞dom和动画节点dom的`z-index`值，这样就可以保证动画播放时依然可以连续点赞，以及点赞后不影响页面中其他元素的显示。

这里要多说一句，在调试层级问题时发现设置`z-index` 有时是无效的，后查明原因是：`z-index` 只能在`position` 属性值为`relative `或`absolute` 或`fixed` 的元素上有效；`z-index` 值只决定同一父元素中的同级子元素的堆叠顺序。

至此，炫酷的点赞动画的实现方案细节和难点都记录在此了，欢迎感兴趣的同学一起讨论交流。
