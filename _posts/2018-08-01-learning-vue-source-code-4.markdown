---
layout:     post
title:      "Vue源码学习4：mergeOptions函数到底在合并什么"
subtitle:   "Learning Vue Source Code 4： What Dose MergeOptions Function Merge"
date:       2018-08-01 12:00:00
author:     "Sandii"
header-img: "img/banner/bg-006.jpg"
catalog: true
tags:
    - vue
---

## merge options

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


## options是什么

先看看options是什么。options就是我们`new Vue`的时候传进来的参数:

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

但`options._isComponent`我没见过，文档里也查不到，所以应该是一个源码内部的属性，并不对用户开放。目前实在不清楚它是干什么用的，而且大概看了一眼`initInternalComponent`，基本也是看不懂，于是先跳过，看`else`分支。


## mergeOptions函数

```
vm.$options = mergeOptions(
  resolveConstructorOptions(vm.constructor),
  options || {},
  vm
)
```

这句大概的意思是，用`mergeOptions`函数处理用户传入的`options`对象，把处理结果赋值给Vue实例对象的`$options`属性。`$options`属性是开放给用户使用的，在API中可以查到。`mergeOptions`是从外部引入的：

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
export function mergeOptions (
  parent: Object, // 接收3个参数
  child: Object,
  vm?: Component
): Object {
  // 各种操作..
  const options = {} //声明一个新对象 (不影响原有对象)
  // 各种操作..
  return options // 把合并好对象的返回出去 (赋给vm.$options)
}
```

说了半天合并options，到底合并什么呢？我们看看这三个参数分别是什么意思：

|函数内|函数外|说明|
|-|-|-|
|parent|resolveConstructorOptions(vm.constructor)|？|
|child|options 或 {}|用户传入的配置项，没传的话就是空对象|
|vm|vm|实例对象|

child和vm都没问题，parent还需要再分析一下。


## resolveConstructorOptions

`resolveConstructorOptions`函数就位于`src/core/instance/init.js`里，看看它是怎么从Vue构造函数上获取`parent`参数:

```
export function resolveConstructorOptions (Ctor: Class<Component>) {
  // 一般来说，Ctor === vm.constructor === Vue
  let options = Ctor.options

  // Ctor也有不是Vue的情况
  // 存在super说明Ctor是Vue的子类
  // 学习过教程我们知道可以用 vue.extends 创建子类
  if (Ctor.super) {
    // 省略..
    // 子类这块逻辑看不太明白，所以也先跳过
  }
  return options
}
```

跳过子类的逻辑后，我们可以认为这个函数只有一句`return Vue.options`。这说明Vue上本身就挂载了一个options属性，在文档上查不到，说明也是源码内部使用的。它是什么时候被挂载上来的我们还不知道，但是却没法跳过它，因为如果不知道parent是什么的话我们就完全没法理解后面的合并操作。

只能先作个弊，回到我们那个简陋的应用，在浏览器控制台里输入一句`Vue.options`:

![](img/content/001.jpg)

这说明`Vue.options`定义了Vue内置功能，比如`component`里有`transition`就意味着我们在模板里可以随时使用`transition`标签；`directive`里有`model`和`show`说明我们可以随时使用`v-show`和`v-model`指令。

## 总结

所以，`mergeOptions`的功能是合并 **用户传入的options** 和 **Vue.options**，作为真正实例化时的配置。

具体的怎么合并又得放到下一篇了。


> 封面图： 槐柏树街 - 2013春 - Sandii
