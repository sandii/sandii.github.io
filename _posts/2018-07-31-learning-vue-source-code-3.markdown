---
layout:     post
title:      "Vue源码学习3：init方法"
subtitle:   "Learning Vue Source Code 3： init method"
date:       2018-07-31 12:00:00
author:     "Sandii"
header-img: "img/banner/bg-006.jpg"
catalog: true
tags:
    - vue
---


## 开始

现在开始依次读下面这几个文件：

```
// src/core/instance/index.js
// src/core/index.js
// src/platforms/web/runtime/index.js
// src/platforms/web/entry-runtime-with-compiler.js
```

从`src/core/instance/index.js`开始，在声明了`构造函数Vue`后有这么几个方法：

```
initMixin(Vue)
stateMixin(Vue)
eventsMixin(Vue)
lifecycleMixin(Vue)
renderMixin(Vue)
```

分别对应：
- src/core/instance/init.js
- src/core/instance/state.js
- src/core/instance/events.js
- src/core/instance/lifecycle.js


## initMixin

`src/core/instance/init.js`输出了`initMixin函数`，目的也很单纯，就是为了给Vue挂载一个实例方法`_init`:

```
export function initMixin (Vue: Class<Component>) {
	Vue.prototype._init = function (options?: Object) {
		// 省略...
	}
}
```

而这个_init方法在实例化的时候是立刻执行的：

```
function Vue (options) {
	this._init(options);
}
```

然而里面的内容我是完全看不懂的…… 分成几个部分一行一行慢慢看吧。

首先是一堆零碎，没事啥好说的：

```
const vm: Component = this	// 将实例对象缓存为变量vm

// a uid
vm._uid = uid++	// 给每一个实例一个独立的_uid

// a flag to avoid this being observed
// 为实例设置一个flag，防止被观察（观察？完全不知道大神你在说什么…）
vm._isVue = true
```

然后就是一段高深的东西了。

## performance

尽管这段代码分为两部分，但一看就是一类的，放在一起研究一下吧…

```
let startTag, endTag
/* istanbul ignore if */
if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
  startTag = `vue-perf-start:${vm._uid}`
  endTag = `vue-perf-end:${vm._uid}`
  mark(startTag)
}

// 省略…

/* istanbul ignore if */
if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
  vm._name = formatComponentName(vm, false)
  mark(endTag)
  measure(`vue ${vm._name} init`, startTag, endTag)
}
```

两个if的判断是一模一样的，说明这两段代码执行的有条件的：
1. 开发环境
1. config.performance
1. mark

开发环境没毛病，config.performance也简单，[文档](https://cn.vuejs.org/v2/api/#performance)说：

> 设置为 true 以在浏览器开发工具的性能/时间线面板中启用对组件初始化、编译、渲染和打补丁的性能追踪。只适用于开发模式和支持 performance.mark API 的浏览器上。

原来是为了追踪组件性能的，也就是控制台里的这种东西了
![](https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1488950633392&di=791ac244d34efbffa8845e49abd307a7&imgtype=0&src=http%3A%2F%2Fa3.att.hudong.com%2F62%2F00%2F01300000060155120062002579414.jpg)




> 封面图： 槐柏树街 - 2013春 - Sandii
