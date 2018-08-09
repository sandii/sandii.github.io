---
layout:     post
title:      "Vue源码学习8： options.data的合并策略"
subtitle:   "Learning Vue Source Code 8: merging strategy for options.data"
date:       2018-08-08 12:00:00
author:     "Sandii"
header-img: "img/pexels/008.jpg"
catalog: true 
tags:
    - vue
---

这篇单独介绍一下options.data的合并策略


```
strats.data = function (
  parentVal: any,
  childVal: any,
  vm?: Component
): ?Function {
  if (!vm) {
    if (childVal && typeof childVal !== 'function') {
      process.env.NODE_ENV !== 'production' && warn(
        'The "data" option should be a function ' +
        'that returns a per-instance value in component ' +
        'definitions.',
        vm
      )
      return parentVal
    }
    return mergeDataOrFn(parentVal, childVal) // 这行貌似是多余的？
  }
  return mergeDataOrFn(parentVal, childVal, vm)
}
```

首先校验非根组件data必须是函数，否则忽略child直接用parent，开发环境会警告。然后使用`mergeDataOrFn()`函数合并data：

```
export function mergeDataOrFn (
  parentVal: any,
  childVal: any,
  vm?: Component
): ?Function {
  if (!vm) {
    // 非根组件，省略...
  } else {
    // 根组件，省略...
  }
}
```

`mergeDataOrFn()`函数主要考虑两种情况，根组件和非根组件。先看非根组件：

```
// in a Vue.extend merge, both should be functions
if (!childVal) {
  return parentVal
}
if (!parentVal) {
  return childVal
}
// 一开始的注释透露出：非根组件调用`mergeOptions()`发生在`Vue.extend()`方法中，参数不传vm。
// 然后的逻辑也很简单，若child和parent二者只存在其一，就直接返回。

// when parentVal & childVal are both present,
// we need to return a function that returns the
// merged result of both functions... no need to
// check if parentVal is a function here because
// it has to be a function to pass previous merges.

// parent和child同时存在，返回一个函数
// 这个函数会使用mergeData函数来合并parent和child的值
// 如果parent和child是函数，执行它们来取得结果
// 这里其实parent肯定是函数，没有必要检验了

return function mergedDataFn () {
  return mergeData(
    typeof childVal === 'function' ? childVal.call(this, this) : childVal,
    typeof parentVal === 'function' ? parentVal.call(this, this) : parentVal
  )
}
```

值得注意的有两点：
1. 合并data后返回一个函数，暂时不执行，现在猜测有可能是为了可以在data中引用props中的值
1. `child.call(this, this)` 第一个`this`是上下文，第二个是参数，所以我们在写options.data的时候可以这么写：

```
props : ['a','b'],
data (vm) {
	return {
		_a : this.a, // 上下文
		_b : vm.b, // 参数
	};
},
```

再看根组件：

```
return function mergedInstanceDataFn () {
  // instance merge
  const instanceData = typeof childVal === 'function'
    ? childVal.call(vm, vm)
    : childVal
  const defaultData = typeof parentVal === 'function'
    ? parentVal.call(vm, vm)
    : parentVal
  if (instanceData) {
    return mergeData(instanceData, defaultData)
  } else {
    return defaultData
  }
}
```

和非根组件基本一样，也是返回函数，只是现在有vm了，可以用vm代替this了。

再看看`mergeData`函数，别忘了它是被延迟执行的：

```
function mergeData (to: Object, from: ?Object): Object {
  if (!from) return to
  let key, toVal, fromVal
  const keys = Object.keys(from)
  for (let i = 0; i < keys.length; i++) {
    key = keys[i]
    toVal = to[key]
    fromVal = from[key]
    if (!hasOwn(to, key)) {
      set(to, key, fromVal)  // parent补充child
    } else if (isPlainObject(toVal) && isPlainObject(fromVal)) { // 双方都是对象，递归合并
      mergeData(toVal, fromVal)
    }
  }
  return to	// 修改child对象
}
```

这才是真正合并data对象的地方，有三个特点：
1. **parent补充child**
1. 双方都是对象，递归合并
1. 合并会修改child对象

最后看一下工具函数set，它来自`src/core/observer/index.js`，一看这个地址就知道水深了，应该是牵涉到Vue的响应式原理，这里就先跳过了。
