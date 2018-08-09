---
layout:     post
title:      "Vue源码学习8： options合并策略"
subtitle:   "Learning Vue Source Code 8: strategies for merging options"
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

第一点和第三点比较简单，上篇已经了解了。这篇主要就是看第二点：`Vue内置属性们策略`，分为六种策略，全都写在本文件中`src/core/util/options.js`：
1. el propsData
1. data
1. 生命周期钩子
1. 资源 components directives filters
1. watch
1. 其他 props methods inject computed


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
if (!child) return parent;
if (parent) return parent.concat(child);
return (Array.isArray(child)) ? child : [child]; 
```

因此逻辑是 `[...parent, ...child]` 把parent和child累加成数组，先parent后child，依次执行。所以我们可以知道：
1. 生命周期钩子本身可以写为数组
1. mixin中的声明周期钩子会先执行


## 资源 components directives filters


## watch
## 其他 props methods inject computed


