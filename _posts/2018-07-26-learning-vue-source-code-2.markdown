---
layout:     post
title:      "Vue源码学习2：构造函数"
subtitle:   "Learning Vue Source Code 2： Constructor"
date:       2018-07-21 12:00:00
author:     "Sandii"
header-img: "img/banner/bg-006.jpg"
catalog: true
tags:
    - vue
---

## 书接上回

我们上一次用script标签引入dist/vue.js，并创建了一个简陋的应用：

```
new Vue({
	el : '#app',
	data : { a : 'hello' },
});
```

现在要给这个简陋的应用增加各种功能该怎么做呢？查看教程和文档我们知道，要调用Vue提供的各种功能有这么几种方法：
1. new Vue()的时候传入各种配置项
1. 调用Vue的静态属性和方法
1. 调用Vue的实例属性和方法

那么我们就可以认为，整套源码主要干了这么的几件事：
1. 声明了一个构造函数Vue
1. 处理实例化时的配置项
1. 为Vue挂载各种静态属性和方法
1. 为Vue实例挂载各种属性和方法


## 背单词

|英文|含义|说明|
|-|-|-|
||||
||||


> 封面图： 槐柏树街 - 2013春 - Sandii
