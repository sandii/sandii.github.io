---
layout:     post
title:      "Vue的contenteditable元素双向绑定"
subtitle:   "Two-way Binding of Vue on Contenteditable"
date:       2018-07-09 12:00:00
author:     "Sandii"
header-img: "img/banner/bg-002.jpg"
catalog: true
tags:
    - Javascript
    - Vue
---

## v-model双向绑定
- vue用内置的`v-model`指令实现`双向绑定`

```
<input type="text" v-model="a" />
// 等价于：
<input 
	type="text"
	:value="a" 
	@input="a = $event.target.value" />
```

|数据流|实现|
|-|-|
|model-view|:value="data"|
|view-model|@input="data = el.value"|


## contenteditable元素双向绑定
- 但用在contenteditable的元素上就不灵了，因为`div`没有`value`属性

```
// 不灵
<div contenteditable v-model="a"><div>
// 等价于
<div 
    contenteditable 
    :value="a" // 问题出在这
    @input="a = $event.target.value"><div>
```

## innerHTML代替value
- 找到原因就好办了，在`div`上用`innerHTML`代替`value`：

|数据流|实现|
|-|-|
|model-view|:v-html="a"|
|view-model|@input="data = el.innerHTML"|

> 修改元素的原生属性最好使用指令，修改innerHTML可以直接使用vue内置指令v-html

```
<div 
    contenteditable 
    v-html="a"
    @input="a = $event.target.innerHTML"><div>
```

- 按下葫芦起了瓢，`value`的问题解决了，但每输入一个字符，光标都会跳回到最开始
- 这是因为数据流`model-view`流动时，给`innerHTML`重新赋值会触发元素重新渲染
- 所以接下来的问题是 **如何阻止输入时重新渲染**
- demo: <https://jsfiddle.net/sandii/k4v2w65t/35/>

## 方案1：自定义指令
1. 增加一个标志位表示是否正在输入，这里放在元素的`dataset`上，因为自定义指令函数中取不到实例对象vm，数据读写通常通过`dataset`进行
1. 用一个自定义指令代替`v-html`，判断如果是输入中，则暂时掐断`model-view`数据流，阻止重新渲染
1. 用`focus / blur`事件切换是否正在输入

```
<div 
    contenteditable 
    v-test="a"		    // 使用自定义指令实现 model-to-view
    data-inputting="0"	// 锁放在元素的dataset上
    @blur="$event.target.dataset.inputting = '0'"// 失焦解锁
    @focus="$event.target.dataset.inputting = '1'"// 聚焦加锁
    @input="a = $event.target.innerHTML"></div>	// v-m 不变

// 自定义指令
directives : {
  	test (el, { value }) {
        // 正在输入 不执行m-v
        if (el.dataset.inputting === '1') return;
        el.innerHTML = value;
    },
},
```

- demo: <https://jsfiddle.net/sandii/k4v2w65t/34/>

## 方案2：watch
1. 整体思路不变，但把`model-view`以及相关判断从自定义指令改在了`watch`上
1. 好处是避免使用自定义指令，从而避免把标志位放在元素的`dataset`上

```
<div 
    contenteditable
    v-html="viewA"
    @blur="inputting = false"// 失焦解锁
    @focus="inputting = true"// 聚焦加锁
    @input="a = $event.target.innerHTML"></div>

data : vm => ({
    a : 'hello',
    viewA : 'hello',
    inputting : false,
}),
watch : {
    a (v) {
        if (this.inputting) return;
        this.viewA = v;
    },
},
```

- demo: <https://jsfiddle.net/sandii/k4v2w65t/42/>

## 背单词

|英文|翻译|
|-|-|
|two-way binding|双向绑定|

## 参考

- <https://github.com/vuejs/vue/issues/47>
- <https://segmentfault.com/a/1190000008261449> eco

> 封面图： 张掖七彩丹霞 - 2015夏 - Sandii
