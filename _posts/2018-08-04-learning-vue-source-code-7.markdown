---
layout:     post
title:      "Vue源码学习7： 合并options的细节"
subtitle:   "Learning Vue Source Code 7: details of merging options"
date:       2018-08-04 12:00:00
author:     "Sandii"
header-img: "img/pexels/007.jpg"
catalog: true 
tags:
    - vue
---

## mergeOptions函数的递归调用

我们发现`mergeOptions`函数内部有递归调用。

```
const extendsFrom = child.extends
if (extendsFrom) {
  parent = mergeOptions(parent, extendsFrom, vm)
}
if (child.mixins) { // mixins是数组，需要遍历一下
  for (let i = 0, l = child.mixins.length; i < l; i++) {
    parent = mergeOptions(parent, child.mixins[i], vm)
  }
}
```

省略掉我们已经看过的和下一步看的代码，可以清楚地看到递归调用是为了把options中的`extends`和`mixins`合并进来。因此我们也就知道了实例化一个组件会产生多次options合并：

|合并顺序|child|parent|
|-|-|-|
|1|options.extends|Ctor.options|
|2|options.mixins|Ctor.options|
|3|options|Ctor.options|

最后我们再看一下上一篇文章中省略的这句：

```
// 之前是子组件名校验
if (typeof child === 'function') {
  child = child.options
}
// 之后是标准化 props inject directives
```

这句话是因为extends属性`可以是一个简单的选项对象或构造函数`（见文档<https://cn.vuejs.org/v2/api/#extends>）。若extends是一个构造函数Ctor，就需要把Ctor.options取出来进行合并。

从这里我们还可以推测出，声明子组件时：

```
const ChildComponent = Vue.extends(options);
ChildComponent.options = mergeOptions(Vue.options, options);
```
但这里还仅仅是猜测，真相还要等到我们看到`Vue.extends`源码时才能知道。到时候也应该就能看懂`resolveConstructorOptions`函数中的代码了。


## 如何合并

经过那么多的准备工作，终于到了具体合并的过程了：

```
const options = {}
let key
for (key in parent) {
  mergeField(key)
}
for (key in child) {
  if (!hasOwn(parent, key)) {
    mergeField(key)
  }
}
function mergeField (key) {
  const strat = strats[key] || defaultStrat
  options[key] = strat(parent[key], child[key], vm, key)
}
return options
```

通过以上代码我们可以发现：
1. 创建了一个新对象
1. 遍历parent和child的各属性，以一定的**策略**合并属性值，赋给新对象
1. 合并并不改变原有的parent和child和对象


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
