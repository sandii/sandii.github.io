---
layout:     post
title:      "Vue中contenteditable元素的双向绑定"
subtitle:   "Two-way binding on contenteditable of Vue"
date:       2018-07-09 12:00:00
author:     "Sandii"
header-img: "img/posts/bg-002.jpg"
catalog: true
tags:
    - Javascript
---

## v-model双向绑定
- vue用内置的`v-model`指令实现`双向绑定`
```
<input type="text" v-model="a" />
```
- 上面的代码等价于：
```
<input 
	type="text"
	:value="a" 
	@input="a = $event.target.value" />
```

|数据流|实现方式|
|-|-|
|model-to-view|将data响应式地赋值给value属性|
|view-to-model|input事件回调中，将value属性赋值给data|


## contenteditable元素的双向绑定
- 但用在contenteditable的元素上就不灵了：
```
// 不灵
<div contenteditable v-model="a"><div>
```
- 因为`div`是没有`value`属性的：
```
// 不灵
<div 
	contenteditable 
	:value="a" 
	@input="a = $event.target.value"><div>
```

## innerHTML代替value
- 找到原因就好办了，用`div`的`innerHTML`属性代替`value`：

|数据流|实现方式|
|-|-|
|model-to-view|将data响应式地赋值给innerHTML|
|view-to-model|input事件回调中，将innerHTML属性赋值给data|

> 修改元素的原生属性最好使用指令，修改innerHTML可以直接使用vue内置指令v-html

```
<div 
	contenteditable 
	v-html="a"
	@input="a = $event.target.innerHTML"><div>
```
- 按下葫芦起了瓢，`value`的问题解决了，但给`innerHTML`重新赋值会触发元素重新渲染，每输入一个字符，光标都会跳回到最开始
- demo: <https://jsfiddle.net/sandii/k4v2w65t/13/>

## 方案1：阻止频繁修改innerHTML
- 不在输入过程中频繁地修改`innerHTML`
```
<div 
    contenteditable 
    class="editor"
    v-test="a"		// 使用自定义指令实现 model-to-view
    data-lock="0"	// 锁放在元素的dataset上
    @blur="$event.target.dataset.lock = '0'"	// 失焦时解锁
    @focus="$event.target.dataset.lock = '1'"	// 聚焦时加锁
    @input="a = $event.target.innerHTML"></div>	// view-to-model 数据流不变

// 自定义指令
directives : {
  	test (el, { value }) {
    	if (el.dataset.lock === '1') return;
    	el.innerHTML = value;
    },
},
```
- demo: <https://jsfiddle.net/sandii/k4v2w65t/31/>

## 背单词

|英文|翻译|
|-|-|
|two-way binding|双向绑定|

## 参考

- <https://stackoverflow.com/questions/47177496/vuejs-2-update-contenteditable-in-component-from-parent-method> user6748331
- <https://stackoverflow.com/questions/46487619/contenteditable-div-append-a-html-element-and-v-model-it-in-vuejs> asemahle
- <https://github.com/vuejs/vue/issues/47>
- <https://segmentfault.com/a/1190000008261449> eco

> 封面图： 张掖七彩丹霞 - 2015夏 - Sandii
