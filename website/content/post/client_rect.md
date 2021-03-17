---
title: "如何获取元素在页面中的绝对位置？"
date: 2018-07-16T11:00:19+08:00
tags: [JavaScript]
draft: false
---

你听说过getBoundingClientRect吗？
<!--more-->

### 从一个坑爹的需求说开去

上周产品小哥哥丢过来一个需求，名曰：沉浸式视频体验，大致内容是一个页面里有几十个视频，用户点击其中一个视频时，该视频自动滑动到屏幕可视区域的顶部开始播放，并暂停其他视频，该视频滑出屏幕可视区域之后要自动暂停。

这个需求有两个关键的技术点：
1. 如何将视频滑动到屏幕可视区域的顶部
2. 如何判断视频滑出了屏幕可视区域

其实这两个技术点有一个共同点，就是需求计算出元素在页面中的绝对位置，也就是指该元素的左上角相对于整张网页左上角的坐标，有两种方法可以计算得到：

#### 1.递归
利用[offsetTop](https://developer.mozilla.org/zh-CN/docs/Web/API/HTMLElement/offsetTop)和[offsetLeft](https://developer.mozilla.org/zh-CN/docs/Web/API/HTMLElement/offsetLeft)可以取到当前元素左上角相对于其`HTMLElement.offsetParent`节点的左边界偏移的像素值，然后再利用[HTMLElement.offsetParent](https://developer.mozilla.org/zh-CN/docs/Web/API/HTMLElement/offsetParent)
可以得到一个指向最近的包含该元素的定位元素。

如果没有定位的元素，则`offsetParent`为最近的`table`,`table cell`或根元素（标准模式下为`html`；`quirks`模式下为`body`）。

利用以上三个属性，写一个递归函数就可以得到当前元素在页面中的绝对位置了：

```
const getElementLeft = element => {
	let actualLeft = element.offsetLeft;
	let current = element.offsetParent;
	while (current !== null){
		actualLeft += current.offsetLeft;
		current = current.offsetParent;
	}
	return actualLeft;
}
const getElementTop = element => {
	let actualTop = element.offsetTop;
	let current = element.offsetParent;
	while (current !== null){
		actualTop += current.offsetTop;
		current = current.offsetParent;
	}
	return actualTop;
}

```
注意：由于在表格和iframe中，`offsetParent`对象未必等于父容器，所以上面的函数对于表格和iframe中的元素不适用。

#### 2. 你听说过getBoundingClientRect吗？
`object.getBoundingClientRect()`的返回值包含了一组只读属性，包括该元素相对于视口左上角位置的`left`、`top`、`right`和`bottom`，以及元素的`width`和`height`，单位为像素，具体含义可参见下图：

![](http://mat1.gtimg.com/www/js/news/wemeet/rect.png)

从上图可以看出，`getBoundingClientRect`得到的值是相对于视口的，而不是绝对的，当视口区域或其他可滚动元素内发生滚动操作时，top和left属性值就会随之立即发生变化。

要获得相对于整个网页左上角定位的属性值，只要给top、left属性值加上当前的滚动位置（通过`window.scrollX`和`window.scrollY`），这样就可以获取与当前的滚动位置无关的值。

### 如何将视频滑动到屏幕可视区域的顶部？
考虑到`getBoundingClientRect`的兼容性较好，且算法复杂度较低，最终我采用了`getBoundingClientRect`方法来实现“将视频滑动到屏幕可视区域的顶部”的功能，使用`window.scroll`来实现页面滚动，使用`setTimeout`增加滚动时的动画，具体实现滚动的函数如下：

```
const autoScroll = (offsetTop, needScrollTop, hasScrollTop) => {
	let _needScrollTop = needScrollTop; // 本次递归时，离终点的距离
	let _hasScrollTop = hasScrollTop; // 本次递归时，已经移动的距离总和
	const speed = 10;
	setTimeout(() => {
		const dist = needScrollTop > 0
			? Math.max(Math.ceil(needScrollTop / speed), 5)
			: Math.min(Math.ceil(needScrollTop / speed), -5);
		_needScrollTop -= dist;
		_hasScrollTop += dist;
		window.scroll(0, offsetTop + _hasScrollTop);
		// 如果移动幅度小于十个像素，直接移动，否则递归调用，实现动画效果
		if (_needScrollTop > speed || _needScrollTop < -speed) {
			this.__onScroll(offsetTop, _needScrollTop, _hasScrollTop);
		} else {
			window.scroll(0, offsetTop + _hasScrollTop + _needScrollTop);
		}
	}, 1);
}

const rect = element.getBoundingClientRect();

const offsetTop = window.pageYOffset || document.documentElement.scrollTop || document.body.scrollTop || window.screenY;

autoScroll(offsetTop, rect.top, 0);

```

### 如何判断视频滑出了屏幕可视区域？
利用`getBoundingClientRect`的返回值同样可以判断元素在屏幕可视区域的曝光和滑出，通过监听页面的滚动事件，在滚动结束时，触发`checkLeave`和`checkExpose`函数，具体代码如下：

```
// 检测滑出可视区域
checkLeave() {
	const element = document.querySelector(`[data-action-id="${this.__domId}"]`);
	if (!element) {
		console.error(`Action: element [data-action-id="${this.__domId}"] not found`);
		return;
	}
	const { top, bottom, left, right } = element.getBoundingClientRect();
	if ((top > getWindowHeight() || bottom < 0
		|| left > getWindowWidth() || right < 0)) {
		this.onLeave(); //onLeave函数中实现具体业务逻辑
	}
}

// 检测真实曝光
checkExpose() {
	const element = document.querySelector(`[data-action-id="${this.__domId}"]`);
	if (!element) {
		console.error(`Action: element [data-action-id="${this.__domId}"] not found`);
		return;
	}
	const { top, bottom, left, right } = element.getBoundingClientRect();
	if (Math.max(0, top) <= Math.min(getWindowHeight(), bottom)
		&& Math.max(0, left) <= Math.min(getWindowWidth(), right)) {
		this.onExpose(); //onExpose函数中实现具体业务逻辑
	}
}

```
可以将这两个事件封装成了一个`<Action>`组件的两个props参数，使用时只需要在需要的元素外包一层`<Action>`父元素，并传入特定的回调函数即可。

BTW，多说一句我实现`document.querySelector`的原理，直接看代码：

```
render() {
	const { children } = this.props;
	return cloneElement(Children.only(children), {
		'data-action-id': this.__domId,
	});
}

```

以上。
