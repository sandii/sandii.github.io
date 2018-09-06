---
layout:     post
title:      "Vue源码学习4： 详解mergeOptions函数（下）"
subtitle:   "Learning Vue Source Code 6: MergeOptions Function Insight 2"
date:       2018-08-11 12:00:00
author:     "Sandii"
header-img: "img/pexels/004.jpg"
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



`strat`是`strategy`的缩写。strat对象就声明在本文件`src/core/util/options.js`。我们看看这个对象是怎么来的：

1. 先从config.optionMergeStrategies中取用户自定义的合并策略

```
const strats = config.optionMergeStrategies
```

config.optionMergeStrategies是开放给用户的API，用法很简单：`Vue.config.optionMergeStrategies.a = (child, parent) => child + 1;`。


2. 然后定义Vue内置属性们的合并策略

```
// 可以分为7组

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

// (7) provide
strats.provide =
```

可以看到，源码是先取`用户自定义策略`，再定义`内置策略`，这保证了内置属性合并不会被用户随意修改。

3. 最后是默认策略
```
// 默认策略就是：child优先
const defaultStrat = function (parentVal: any, childVal: any): any {
  return childVal === undefined
    ? parentVal
    : childVal
}
```

## data（重点）

一上来先说最重要的data合并策略。

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
  return to // 修改child对象
}
```

这才是真正合并data对象的地方，有三个特点：
1. **parent补充child**
1. 双方都是对象，递归合并
1. 合并会修改child对象

其中的工具函数set，它来自`src/core/observer/index.js`，一看这个地址就知道水深了，应该是牵涉到Vue的响应式原理，这里就先跳过了。


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
provide的处理和data一样。


## 总结

到这里，整个`src/core/util/options.js`文件我们就基本上看完了，最后还有一个`export function resolveAsset`现在还不知道什么干什么用的。可以看到这个文件其实就是`mergeOptions()`函数及其内部主要的操作：
1. 检验子组件名
1. 标准化props inject directives
1. 合并options

合并options的具体过程也分为三步：
1. parent = mergeOptions(parent, child.extends)
1. parent = mergeOptions(parent, child.mixins)
1. 逐个属性按策略合并

合并策略分别有：//todo
|key|strategy|
|-|-|
|el propsData||
|生命周期钩子||
|资源 components directives filters||
|watch||
|其他 props methods inject computed||
