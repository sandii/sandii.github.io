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
但这个`options._isComponent`我没见过，文档里也查不到，所以应该是一个内部的属性，并不对用户开放。目前实在不清楚它是干什么用的，只能先往下看。`if-else`里的`initInternalComponent` `mergeOptions` `resolveConstructorOptions`三个方法。

## initInternalComponent

这个

```
export function initInternalComponent (vm: Component, options: InternalComponentOptions) {
  const opts = vm.$options = Object.create(vm.constructor.options)
  // doing this because it's faster than dynamic enumeration.
  const parentVnode = options._parentVnode
  opts.parent = options.parent
  opts._parentVnode = parentVnode

  const vnodeComponentOptions = parentVnode.componentOptions
  opts.propsData = vnodeComponentOptions.propsData
  opts._parentListeners = vnodeComponentOptions.listeners
  opts._renderChildren = vnodeComponentOptions.children
  opts._componentTag = vnodeComponentOptions.tag

  if (options.render) {
    opts.render = options.render
    opts.staticRenderFns = options.staticRenderFns
  }
}
```


> 封面图： 槐柏树街 - 2013春 - Sandii
