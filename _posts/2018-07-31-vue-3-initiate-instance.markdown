---
layout:     post
title:      "Vue源码学习3：实例初始化"
subtitle:   "Learning Vue Source Code 3: Initiate Instance"
date:       2018-07-31 12:00:00
author:     "Sandii"
header-img: "img/pexels/003.jpg"
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


## Vue实例初始化

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

然而里面的内容我是完全看不懂的…… 分成几个部分一行一行慢慢看吧。首先是一堆零碎，没事啥好说的：

```
const vm: Component = this  // 将实例对象缓存为变量vm

// a uid
vm._uid = uid++ // 给每一个实例一个独立的_uid

// a flag to avoid this being observed
// 为实例设置一个flag，防止被观察（观察？完全不知道大神你在说什么…）
vm._isVue = true
```

然后就是一段看不懂的东西了。


## 性能监控

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

开发环境没毛病，`config.performance`也简单，[文档](https://cn.vuejs.org/v2/api/#performance)说：

> 设置为 true 以在浏览器开发工具的性能/时间线面板中启用对组件初始化、编译、渲染和打补丁的性能追踪。只适用于开发模式和支持 performance.mark API 的浏览器上。

原来是为了追踪组件性能的，也就是chrome后台里的performance页签：
![](img/content/2018-07-13-01.jpg)

最后看`mark`，这是一个函数，来源于`src/core/util/perf.js`，这个文件很简明，就是输出`mark`和`measure`两个函数：

```
import { inBrowser } from './env'
// 判断是否是浏览器环境，判断的方法也很简单，就是看有没有window对象
// export const inBrowser = typeof window !== 'undefined'

// 输出这两个变量
export let mark
export let measure

// 如果不符合下面的这两个if的条件，上面的两个输出都为undefined
if (process.env.NODE_ENV !== 'production') {  // 开发环境
  
  // 声明变量perf，若下面的条件都成立，perf === window.performance
  const perf = inBrowser && window.performance 
  /* istanbul ignore if */

  // 浏览器环境 且 window.performance的API可用
  if (
    perf &&   // 
    perf.mark &&
    perf.measure &&
    perf.clearMarks &&
    perf.clearMeasures
  ) {

    // 输出的mark和measure两个函数都是用于调用window.performance的API
    mark = tag => perf.mark(tag)
    measure = (name, startTag, endTag) => {
      perf.measure(name, startTag, endTag)
      perf.clearMarks(startTag)
      perf.clearMarks(endTag)
      perf.clearMeasures(name)
    }
  }
}
```

所以，init里的这两段代码是通过调用window.performanceAPI来对vue组件进行性能监控。


## 监控内容

被监控的代码是：

```
// a flag to avoid this being observed
vm._isVue = true
// merge options
if (options && options._isComponent) {
  // optimize internal component instantiation
  // since dynamic options merging is pretty slow, and none of the
  // internal component options needs special treatment.
  initInternalComponent(vm, options)
} else {
  vm.$options = mergeOptions(
    resolveConstructorOptions(vm.constructor),
    options || {},
    vm
  )
}
/* istanbul ignore else */
if (process.env.NODE_ENV !== 'production') {
  initProxy(vm)
} else {
  vm._renderProxy = vm
}
// expose real self
vm._self = vm
initLifecycle(vm)
initEvents(vm)
initRender(vm)
callHook(vm, 'beforeCreate')
initInjections(vm) // resolve injections before data/props
initState(vm)
initProvide(vm) // resolve provide after data/props
callHook(vm, 'created')
```

他们具体是干嘛用的还不知道，但可以猜到的是这些代码就是Vue实例初始化的具体过程了，因为监控了它们就相当于监控了组件。而且我有一种不好的预感，可能要看懂这里的每一行我都得写一篇文章的了orz……同时我也有一种不错的预感，可能看完了这些，Vue的全貌基本也就掌握了。希望如此吧。


## 组件挂载

监控完成后，执行：

```
if (vm.$options.el) {
  vm.$mount(vm.$options.el)
}
```
尽管目前还不能确定`$mount`和`$options.el`的含义，但基本可以猜到，这一句的作用是是把vue组件挂载到dom中去。


## 总结

- `src/core/instance/index.js`声明了`构造函数Vue`
- 实例化时执行`_init方法`，而这个方法来源于`src/core/instance/init.js`
- 初始化执行了一大堆不知道什么操作，并用window.performance监控了全程（所以很重要）
- 初始化的最后是把组件挂载到DOM中去
