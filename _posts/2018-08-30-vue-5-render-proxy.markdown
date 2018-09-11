---
layout:     post
title:      "Vue源码学习10： renderProxy"
subtitle:   "Learning Vue Source Code 10: renderProxy"
date:       2018-08-30 12:00:00
author:     "Sandii"
header-img: "img/pexels/007.jpg"
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

纳尼，开发环境和生产环境执行的操作居然不一样。按照经验，我们之前看到`if (process.env.NODE_ENV !== 'production')`这一句后面一般都是跟着对用户的某些输入的校验，并不会影响代码的主要逻辑。因此，`initProxy()`函数的作用也应该是为实例对象挂载一个`_renderProxy`属性。

打开`src/core/instance/proxy.js`，整个文件只有几十行，只输出了一个`initProxy()`函数：

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

果然，不管开发环境还是生产环境都是为了给`实例对象`增添一个`_renderProxy`属性。整体逻辑可以改写为：

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
1. 若是生产环境或不支持原生Proxy，直接挂vm。


## renderProxy 渲染代理

我们知道，`Proxy对象`用于拦截针对目标对象的所有操作，因此，对`_renderProxy属性`的操作将被拦截。具体的拦截方式我们看`handlers`对象：

```
const options = vm.$options
const handlers = options.render && options.render._withStripped
    ? getHandler
    : hasHandler
```

瞎了，看不懂。`options.render`没有问题，就是用户传入的render方法。但是`options.render._withStripped`是个毛线球啊。先不管它，可以看到`_withStripped`存在与否决定了对`_renderProxy`的拦截方式的不同。存在是`getHandler`，不存在是`hasHandler`。

```
const getHandler = {
    get (target, key) {
        if (typeof key === 'string' && !(key in target)) {
            warnNonPresent(target, key)
        }
        return target[key]
    }
}
```

先看`getHandler`：
1. 拦截对_renderProxy的`**读**操作`
1. 若访问的属性不存在就报警
1. 不改变默认返回值

```
const warnNonPresent = (target, key) => {
    warn(
        `Property or method "${key}" is not defined on the instance but ` +
        'referenced during render. Make sure that this property is reactive, ' +
        'either in the data option, or for class-based components, by ' +
        'initializing the property. ' +
        'See: https://vuejs.org/v2/guide/reactivity.html#Declaring-Reactive-Properties.',
        target
    )
}
```

仔细看了一眼报警内容：在渲染过程中调用的属性或方法在实例对象上不存在……这句话很有价值。它告诉我们在实例渲染时，**渲染函数会从`vm._renderProxy`对象上读取所需要的数据，一般来说他指向的是实例对象，也就是说直接从实例对象上读取数据用于渲染。但在开发环境，且支持Proxy的情况下，程序会给这个操作增加一层校验：渲染所需数据不存在时报警。**

```
const hasHandler = {
  has (target, key) {
    const has = key in target
    const isAllowed = allowedGlobals(key) || key.charAt(0) === '_'
    if (!has && !isAllowed) {
      warnNonPresent(target, key)
    }
    return has || !isAllowed
  }
}
```
再看`hasHandler`。
1. 顾名思义，它拦截的是枚举操作。
1. 不存在且不特许时，报警

那么Vue特许什么样的属性名呢？也就是渲染时引用什么样的数据，就算不存在也不报警呢？
1. js语言的内置全局变量
1. 以下划线开头的数据

```
const allowedGlobals = makeMap(
    'Infinity,undefined,NaN,isFinite,isNaN,' +
    'parseFloat,parseInt,decodeURI,decodeURIComponent,encodeURI,encodeURIComponent,' +
    'Math,Number,Date,Array,Object,Boolean,String,RegExp,Map,Set,JSON,Intl,' +
    'require' // for Webpack/Browserify
);
```

比较有意思的是这个拦截枚举的返回值`return has || !isAllowed`。数据存在时返回`true`，不存在返回是否非特许。这个微妙的返回值在后面估计会有作用。



## 校验自定义快捷键名

从API中我们知道，Vue有着一系列修饰符可以作为绑定事件的辅助语法糖，十分贴心：

```
@click.stop="handleClick"
// 等价于在handleClick中调用e.stopPropagation()
```

可以使用`config.keyCodes`自定义这样的修饰符。

```
Vue.config.keyCodes = {
  f1 : 112,
  'arrow-up' : [38, 87],  // kebab-case instead of camelCase
};

// 调用
@keyup.f1="handleF1"
@keydown.arrow-up="handleArrowUp"
```

而这个文件里最后一段代码就是用于校验自定义的修饰符不能覆盖Vue内置的修饰符。

```
if (hasProxy) {
  const isBuiltInModifier = makeMap('stop,prevent,self,ctrl,shift,alt,meta,exact')
  config.keyCodes = new Proxy(config.keyCodes, {
    set (target, key, value) {
      if (isBuiltInModifier(key)) {
        warn(`Avoid overwriting built-in modifier in config.keyCodes: .${key}`)
        return false
      } else {
        target[key] = value
        return true
      }
    }
  })
}
```

这段代码的外面还套了一层`if (process.env.NODE_ENV !== 'production')`，因此同样的，这个校验只会在开发环境且支持Proxy的情况下起作用。


## 总结

`initProxy方法`或者说`src/core/instance/proxy.js`这个文件的作用是在开发环境且支持Proxy的情况下利用Proxy来进行校验：
1. 校验render函数是否引用了vm上不存在数据或非特许的数据
1. 校验自定义的快捷键名是否和Vue内置的快捷键修饰符重名

所以叫他们要叫做Proxy啊。

其中渲染校验和渲染函数紧密相关，而渲染函数我们还并不熟悉。可以知道的是，渲染函数会从`vm._renderProxy`对象上读取所需要的数据，一般来说他指向的是实例对象。但在开发环境Vue利用Proxy增加一层校验：渲染所需数据不存在时报警。

但目前我们仍然有一块关键的知识并不明确。`options.render._withStripped`是什么，为什么他存在时拦截读取，不存在时拦截枚举呢。


## 背单词

|英文|翻译|
|-|-|-|
|||
|||
|||
|||


