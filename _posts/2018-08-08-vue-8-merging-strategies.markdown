---
layout:     post
title:      "Vue源码学习8： options合并策略"
subtitle:   "Learning Vue Source Code 8: options merging strategies"
date:       2018-08-08 12:00:00
author:     "Sandii"
header-img: "img/pexels/008.jpg"
catalog: true 
tags:
    - vue
---

上一篇中我们说过

## 合并策略

```
function mergeField (key) {
  const strat = strats[key] || defaultStrat
  options[key] = strat(parent[key], child[key], vm, key)
}
```
从上面代码我们可以发现：
- 每合并一个属性，都会去找这个属性所对应的合并策略。
- 若该属性的策略不存在，就使用默认策略。
- 每个策略都是一个函数，接收parent和child的属性值，返回合并后的值。



`strat`是`strategy`的缩写。strat对象就声明在本文件`src/core/util/options`。我们看看这个对象是怎么来的：

1. 先从config.optionMergeStrategies中取用户自定义的合并策略

```
const strats = config.optionMergeStrategies
```

config.optionMergeStrategies是开放给用户的API，用法很简单：`Vue.config.optionMergeStrategies.a = (child, parent) => child + 1;`。


2. 然后定义Vue内置属性们的合并策略

```
// 可以分为六组

// (1) el和propsData策略一样
strats.el = strats.propsData = 

// (2) data
strats.data = 

// (3) 生命周期钩子
LIFECYCLE_HOOKS.forEach(hook => {
  strats[hook] = mergeHook
})

// (4) 资源 components directives filters
ASSET_TYPES.forEach(function (type) {
  strats[type + 's'] = mergeAssets
})

// (5) watch 
strats.watch = 

// (6) props methods inject computed
strats.props =
strats.methods =
strats.inject =
strats.computed =
```

可以看到，源码是先取`用户自定义策略`，再定义`内置策略`，这保证了内置属性合并不会被用户随意修改。

3. 最后是默认策略
```
// 默认策略就是 child优先
const defaultStrat = function (parentVal: any, childVal: any): any {
  return childVal === undefined
    ? parentVal
    : childVal
}
```

今天到此为止，一篇文章再详解Vue内置属性们的合并策略。
