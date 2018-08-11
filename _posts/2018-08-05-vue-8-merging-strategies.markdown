---
layout:     post
title:      "Vue源码学习8： options合并策略"
subtitle:   "Learning Vue Source Code 8: Strategies for Merging Options"
date:       2018-08-05 12:00:00
author:     "Sandii"
header-img: "img/pexels/008.jpg"
catalog: true 
tags:
    - vue
---

上一篇中我们已经知道了，合并options的最终步骤就是针对不同属性采用不同的合并策略。

```
function mergeField (key) {
  const strat = strats[key] || defaultStrat
  options[key] = strat(parent[key], child[key], vm, key)
}
```

合并策略分为三类：
1. 用户自定义策略 `config.optionMergeStrategies`
1. Vue内置属性们策略
1. 默认策略（找不到对应策略时使用，child优先）

第一点和第三点比较简单，上篇已经了解了。这篇主要就是看第二点：`Vue内置属性们策略`，分为7种策略，全都写在本文件中`src/core/util/options.js`：
1. el propsData
1. data
1. 生命周期钩子
1. 资源 components directives filters
1. watch
1. 其他 props methods inject computed
1. provide


## el propsData

```
if (process.env.NODE_ENV !== 'production') {
  strats.el = strats.propsData = function (parent, child, vm, key) {
    if (!vm) {
      warn(
        `option "${key}" can only be used during instance ` +
        'creation with the `new` keyword.'
      )
    }
    return defaultStrat(parent, child)
  }
}
```

很简单，el和propsData采用`默认策略`。

但开发环境中会进行一项检验，若`vm`参数为空发出警告：

> option ${key} key can only be used during instance creation with the new keyword
> 翻译：该属性只能用在用new创建出的实例中

确实如此，文档中已经告诉我们`el`和`propsData`属性只能用在根组件里，子组件是不能用的。这里就是校验这个的。

从判断条件`if (!vm)`可以推测出，**源码通过参数`vm`是否存在来判断当前实例是否根是组件**。在`Vue.prototype._init()`中调用`mergeOptions()`确实传入了当前实例对象`vm`。但我们目前还没见过未传入vm的情况，但可以推测出子组件实例化时调用`mergeOptions()`是不传`vm`的。这个等到以后碰到的时候再做验证。

## data
比较复杂，下一篇专门写这块。

## 生命周期钩子

```
// 所有生命周期钩子的合并策略都一样，mergeHook函数
LIFECYCLE_HOOKS.forEach(hook => {
  strats[hook] = mergeHook
})

// 都有哪些生命周期钩子呢
// src/shared/constants.js 
export const LIFECYCLE_HOOKS = [
  'beforeCreate',
  'created',
  'beforeMount',
  'mounted',
  'beforeUpdate',
  'updated',
  'beforeDestroy',
  'destroyed',
  'activated',
  'deactivated',
  'errorCaptured'
]

// 这是具体策略，合并后生成数组
// Hooks and props are merged as arrays.
function mergeHook (
  parentVal: ?Array<Function>,
  childVal: ?Function | ?Array<Function>
): ?Array<Function> {
  return childVal
    ? parentVal
      ? parentVal.concat(childVal)
      : Array.isArray(childVal)
        ? childVal
        : [childVal]
    : parentVal
}
```

逻辑比较简单，翻译一下这个厉害的三元运算符三连击：

```
// 若parent不存在，返回child，要保证是数组
if (!parent) return (Array.isArray(child)) ? child : [child];

// 若child不存在，返回parent，parent只要存在就肯定是数组
if (!child) return parent;

// parent和child都存在，连接起来
return parent.concat(child);
```

简而言之逻辑是 `[...parent, ...child]` ，所以我们可以知道：
1. 生命周期钩子本身可以写为数组
1. mixin中的声明周期钩子会先执行


## 资源 components directives filters

`components directives filters`统称为`资源asset`，可能是因为比较适合作为第三方插件。

```
export const ASSET_TYPES = [
  'component',
  'directive',
  'filter'
]
ASSET_TYPES.forEach(function (type) {
  strats[type + 's'] = mergeAssets
})
function mergeAssets (
  parentVal: ?Object,
  childVal: ?Object,
  vm?: Component,
  key: string
): Object {
  const res = Object.create(parentVal || null)
  if (childVal) {
    process.env.NODE_ENV !== 'production' && assertObjectType(key, childVal, vm)
    return extend(res, childVal)
  } else {
    return res
  }
}
```

逻辑也很简单：
1. 新建一个对象 （合并不影响child和parent的原对象）
1. 把`__proto__`指向parent （使用`v-show`或`<router-view>`，是从原型链上取）
1. 从child上浅克隆属性 （child会覆盖parent）

最后看一下`assertObjectType`工具函数：

```
function assertObjectType (name: string, value: any, vm: ?Component) {
  if (!isPlainObject(value)) {
    warn(
      `Invalid value for option "${name}": expected an Object, ` +
      `but got ${toRawType(value)}.`,
      vm
    )
  }
}
```

也比较简单，就是校验一下，child必须是普通对象，封装了警告信息。


## watch

```
strats.watch = function (
  parentVal: ?Object,
  childVal: ?Object,
  vm?: Component,
  key: string
): ?Object {
  // 1. 先屏蔽掉火狐的Object.prototype.watch的影响
  // src/core/util/env.js
  // export const nativeWatch = ({}).watch

  // work around Firefox's Object.prototype.watch...
  if (parentVal === nativeWatch) parentVal = undefined
  if (childVal === nativeWatch) childVal = undefined
  /* istanbul ignore if */

  // 2. 没有child用parent 没有parent用child
  if (!childVal) return Object.create(parentVal || null)
  if (process.env.NODE_ENV !== 'production') {
    assertObjectType(key, childVal, vm)
  }
  if (!parentVal) return childVal

  // 新建一个对象，浅拷贝parent
  // 把child连接上去 [...parent, ...child]
  const ret = {}
  extend(ret, parentVal)
  for (const key in childVal) {
    let parent = ret[key]
    const child = childVal[key]
    if (parent && !Array.isArray(parent)) {
      parent = [parent]
    }
    ret[key] = parent
      ? parent.concat(child)
      : Array.isArray(child) ? child : [child]
  }
  return ret
}
```

对`watch`的处理类似于`hook`，把每个属性合并为数组，依次执行。


## 其他 props methods inject computed

```
strats.props =
strats.methods =
strats.inject =
strats.computed = function (
  parentVal: ?Object,
  childVal: ?Object,
  vm?: Component,
  key: string
): ?Object {
  if (childVal && process.env.NODE_ENV !== 'production') {
    assertObjectType(key, childVal, vm) // 校验child是普通对象
  }
  if (!parentVal) return childVal // 没有parent直接用child
  const ret = Object.create(null) // 新建空对象
  extend(ret, parentVal) // 浅拷贝parent
  if (childVal) extend(ret, childVal) // 浅拷贝child
  return ret // 返回新对象
}
```

逻辑是：
1. 新建对象
1. 浅拷贝parent
1. 浅拷贝child，覆盖parent


## provide
provide的处理和data相关，也放到下一篇。

## 总结

除了data和provide其他都看完了，初步总结一下：

|key|strategy|
|-|-|
|el propsData||
|生命周期钩子||
|资源 components directives filters||
|watch||
|其他 props methods inject computed||


