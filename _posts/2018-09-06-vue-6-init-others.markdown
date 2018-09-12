---
layout:     post
title:      "Vue源码学习6： 其他初始化"
subtitle:   "Learning Vue Source Code 10: other initiations"
date:       2018-09-06 12:00:00
author:     "Sandii"
header-img: "img/pexels/008.jpg"
catalog: true 
tags:
    - vue
---

## 

继续看`Vue.protottype.init()`函数：

```
// expose real self
vm._self = vm
```

先把实例对象挂在自己的`_self`属性上。然后是一系列的初始化函数。这些初始化操作有些是在`beforeCreate`之前有的是在`created`之前。相关的函数按各自的函数名分布在当前目录中，加上`index.js`（创建构造函数）和`init.js`，目录`src/core/instance/`的基本内容也就是这些，而Vue组件的实例化的操作也就是这些了。

```
initLifecycle(vm)
initEvents(vm)
initRender(vm)
callHook(vm, 'beforeCreate')
initInjections(vm) // resolve injections before data/props
initState(vm)
initProvide(vm) // resolve provide after data/props
callHook(vm, 'created')
```


## callHook 调用声明周期钩子

```
export function callHook (vm: Component, hook: string) {
  // #7573 disable dep collection when invoking lifecycle hooks
  pushTarget()
  const handlers = vm.$options[hook]
  if (handlers) {
    for (let i = 0, j = handlers.length; i < j; i++) {
      try {
        handlers[i].call(vm)
      } catch (e) {
        handleError(e, vm, `${hook} hook`)
      }
    }
  }
  if (vm._hasHookEvent) {
    vm.$emit('hook:' + hook)
  }
  popTarget()
}
```

所有Vue的用户一定都对`生命周期钩子函数`非常熟悉，它的调用也非常简单。
1. 传入`vm`和`钩子名`，在`vm.$options`上找到相应的钩子，看过`mergeOptions`之后我们知道钩子函数会被处理成数组，循环调用，以vm作为上下文
1. 调用外部加一层`try/catch`，用`handleError()`函数处理可能的异常
1. 在本实例内vm内广播钩子事件`vm.$emit('hook:' + hook)`

函数开始和结尾的`pushTarget()`和`popTarget()`一看就是一对的，根据注释我们知道它是为了在调用钩子函数时暂时关闭**依赖收集**。这属于Vue数据响应式相关的功能，我们这里暂时跳过。

最后再看一眼`handleError()`函数就可以了：

```

```


## initLifecycle 初始化声明周期
## initEvents 初始化事件
## initRender 初始化渲染
## initInjections 初始化注入
## initState 初始化状态
## initProvide 初始化依赖



## 总结


## 背单词

|英文|翻译|
|-|-|-|
|||
|||
|||
|||


