---
layout:     post
title:      "Vue源码学习10： renderProxy"
subtitle:   "Learning Vue Source Code 10: renderProxy"
date:       2018-08-09 12:00:00
author:     "Sandii"
header-img: "img/pexels/010.jpg"
catalog: true 
tags:
    - vue
---

## initProxy要干什么？

费了好大劲终于把`mergeOptions()`看完了。我们继续读`Vue.prototype._init()`方法。

```
if (process.env.NODE_ENV !== 'production') {
  initProxy(vm)
} else {
  vm._renderProxy = vm
}
```

纳尼，开发环境和生产环境执行的操作居然不一样，不科学啊。按照经验，我们之前看到`if (process.env.NODE_ENV !== 'production')`这一句后面一般都是跟着对用户的某些输入的校验，并不会影响代码的主要逻辑。看看这次是否还是这样。`initProxy()`函数来源于`src/core/instance/proxy.js`，整个文件不过几十行，目的也很明确，就是为了输出`initProxy()`方法：

```
let initProxy
// ...
initProxy = function initProxy (vm) {
    //...
}
export { initProxy }
```
那么`initProxy()`函数究竟干了什么呢？

```
if (hasProxy) {
    // ...
    vm._renderProxy = new Proxy(vm, handlers)
} else {
    vm._renderProxy = vm;
}
```

看来我们的经验没有错。不管开发环境还是生产环境都是为了给`实例对象`增添一个`_renderProxy`属性。逻辑是这样的：

```
vm._renderProxy = process.env.NODE_ENV !== 'production' && hasProxy 
    ? new Proxy(vm, handlers) 
    : vm;

```

那么hasProxy的意思也很简单，就是当前环境是否支持原生Proxy接口：

```
const hasProxy =
    typeof Proxy !== 'undefined' &&
    Proxy.toString().match(/native code/)
```

现在我们可以回答小标题的问题了`initProxy要干什么？`：
1. 为vm挂载一个`_renderProxy属性`（虽然不知道是干嘛用的）
1. 若是开发环境且支持原生Proxy接口，`_renderProxy属性`挂一个`Proxy对象`
1. 若是生产环境或不支持原生Proxy，挂vm。


## 校验什么？

既然经验管用，那么我们继续按照经验来推测：这个`new Proxy(vm, handlers)`是用于校验某项输入的。我们来逐步验证一下这次猜的对不对啊。

现在唯一的线索就是`handlers`变量了：

```
const options = vm.$options
const handlers = options.render && options.render._withStripped
    ? getHandler
    : hasHandler
```

瞎了，看不懂。`options.render`没有问题，就是用户传入的render方法。但是`options.render._withStripped`是个毛线球啊。
