---
layout:     post
title:      "Vue源码学习4："
subtitle:   "Learning Vue Source Code 4： "
date:       2018-08-01 12:00:00
author:     "Sandii"
header-img: "img/banner/bg-006.jpg"
catalog: true
tags:
    - vue
---

## 合并options

我们开始一行一行分析被监控的代码究竟是什么。从这一句开始。

```
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
```

先把英文注释翻译一下：

> 合并options
> 优化内部组件实例化
> 因为动态地去合并options实在太慢了
> 且所有内部组件的options都不需要特殊处理
> （意思就是所有组件的options都可以统一处理）

虽然每个词都能看懂，但还是不明白是什么意思，汗。上一篇文章说过，每一句可能都得写一篇文章，我觉得我的预感是很准的。

先看看options是什么。options实际就是我们`new Vue`的时候传进来的参数:

```
// 简陋的应用
new Vue({
  el : '#app',
  data : { a : 'hello world' },
});
```

此时options就是（也就是我们最熟悉的API的部分）：
```
{
  el : '#app',
  data : { a : 'hello world' },
}
```

但这个`options._isComponent`我没见过，文档里也查不到，所以应该是一个源码内部的属性，并不对用户开放。目前实在不清楚它是干什么用的，而且大概看了一眼`initInternalComponent`，基本也是看不懂，于是先跳过，看`else`分支：

```
vm.$options = mergeOptions(
  resolveConstructorOptions(vm.constructor),
  options || {},
  vm
)
```

这句大概的意思是，用`mergeOptions`函数处理用户传入的`options`对象，把处理结果赋值给Vue实例对象的`$options`属性。这个属性是开放给用户使用的，在API中可以查到。那么我们看一下`mergeOptions`函数是怎么处理options的。


## mergeOptions

`mergeOptions`是从外部引入的

```
import { extend, mergeOptions, formatComponentName } from '../util/index'
```

打开`src/core/util/index.js`，发现是个工具大汇总，又只能靠猜了啊……

```
export * from 'shared/util'
export * from './lang'
export * from './env'
export * from './options'
export * from './debug'
export * from './props'
export * from './error'
export * from './next-tick'
export { defineReactive } from '../observer/index'
```

我猜是`options`，打开`src/core/util/options.js`，猜对了~
但是400多行的代码，加上各种外部引入的东西，估计够我看一阵子了。

```

```




## resolveConstructorOptions




> 封面图： 槐柏树街 - 2013春 - Sandii
