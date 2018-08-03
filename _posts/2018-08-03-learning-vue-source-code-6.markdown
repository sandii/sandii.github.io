---
layout:     post
title:      "Vue源码学习6：标准化options"
subtitle:   "Learning Vue Source Code 5: normalize options"
date:       2018-08-03 12:00:00
author:     "Sandii"
header-img: "img/pexels/006.jpg"
catalog: true
tags:
    - vue
---

校验完child.components命名规范。接下来先跳过这一句，后面再说：

```
if (typeof child === 'function') {
  child = child.options
}
```

接下来是这三行，三个函数，从函数名上来猜是normalize(标准化) props inject directive

```
normalizeProps(child, vm)
normalizeInject(child, vm)
normalizeDirectives(child)
```


## 为什么要标准化

官方文档告诉我们`props` `inject` `directive`这三个选项可兼容多种不同的写法。标准化就是把这多种写法统一成一种，便于后面处理。比如`props`可以有三种写法:

```
// 1. 数组
props : ['a', 'b'],

// 2. 对象，属性值是type
props : {
  c : Number,
  d : String,
},

// 3. 对象，完整写法，除了type之外还可以有很多其他功能
props : {
  e : { type : Number, default : 0 },
  f : { type : String, default : '', require : true },
},
```

标准化过后，上面的abcd转化为：

```
props : {
  a : { type : null },
  b : { type : null },
  c : { type : Number },
  d : { type : String },
},
```

三个标准化函数就在本文件`src/core/util/opions.js`中，我们分别看一遍。


## 标准化props

```
function normalizeProps (options: Object, vm: ?Component) {
  const props = options.props
  if (!props) return
  const res = {}
  let i, val, name
  if (Array.isArray(props)) {	// 1. 处理数组
    i = props.length
    while (i--) {
      val = props[i]
      if (typeof val === 'string') {
        name = camelize(val)
        res[name] = { type: null } // 处理
      } else if (process.env.NODE_ENV !== 'production') {
        warn('props must be strings when using array syntax.')
      }
    }
  } else if (isPlainObject(props)) {
    for (const key in props) {
      val = props[key]
      name = camelize(key)
      res[name] = isPlainObject(val) 
        ? val               // 3. 不用处理
        : { type: val }     // 2. 处理
    }
  } else if (process.env.NODE_ENV !== 'production') { // 写法不对 发警告
    warn(
      `Invalid value for option "props": expected an Array or an Object, ` +
      `but got ${toRawType(props)}.`,
      vm
    )
  }
  options.props = res	// 覆盖原有的props
}
```

逻辑还是比较直观的，只需要再看一下几个工具函数，都在`src/shared/util.js`。

## camelize 驼峰化

```
/**
 * Camelize a hyphen-delimited string.
 */
const camelizeRE = /-(\w)/g
export const camelize = cached((str: string): string => {
  return str.replace(camelizeRE, (_, c) => c ? c.toUpperCase() : '')
})
```

逻辑很清楚，就是`kebab-case`转`camelCase`，官方文档里谈过这个问题，要求用户：

```
// js里用camelCase
props: ['postTitle'],
template: '<h3>{{ postTitle }}</h3>'

<!-- template里用kebab-case -->
<blog-post post-title="hello!"></blog-post>
```

但是这块又引出了另一个工具函数`cached`。


## cached

```
export function cached<F: Function> (fn: F): F {
  const cache = Object.create(null)
  return (function cachedFn (str: string) {
    const hit = cache[str]
    return hit || (cache[str] = fn(str))
  }: any)
}
```

`cached函数`为普通函数增加缓存功能。目前猜测可能是源码某处需要以同样的参数频繁调用`camelize`，所以加上缓存优化。


## isPlainObject 和 toRawType

```
// 判断数据类型都得调用Object.prototype.toString
const _toString = Object.prototype.toString
export function isPlainObject (obj: any): boolean {
  return _toString.call(obj) === '[object Object]'
}
export function toRawType (value: any): string {
  return _toString.call(value).slice(8, -1) // 提取后面那个词
}
```


## 标准化inject

inject和props很类似，但关键是from属性：

```
// 1. 数组
inject : ['a','b'],
// 2. 对象，属性值是from
inject : { c : '_c', d : '_d' },
inject : {
  e : { default ： '' },  // 3. 缺少from 
  f : { from : 'f', default : 0 }, // 4. 完整写法
}
```

```
function normalizeInject (options: Object, vm: ?Component) {
  const inject = options.inject
  if (!inject) return
  const normalized = options.inject = {}
  if (Array.isArray(inject)) { // 1. 处理数组
    for (let i = 0; i < inject.length; i++) {
      normalized[inject[i]] = { from: inject[i] }
    }
  } else if (isPlainObject(inject)) {
    for (const key in inject) {
      const val = inject[key]
      normalized[key] = isPlainObject(val)
        ? extend({ from: key }, val) // 3 & 4. 补足from字段
        : { from: val }              // 2. 把属性值设为from
    }
  } else if (process.env.NODE_ENV !== 'production') {
    warn(
      `Invalid value for option "inject": expected an Array or an Object, ` +
      `but got ${toRawType(inject)}.`,
      vm
    )
  }
}
```


## 标准化directives

directive比之前两个简单：

```
function normalizeDirectives (options: Object) {
  const dirs = options.directives
  if (dirs) {
    for (const key in dirs) {
      const def = dirs[key]
      if (typeof def === 'function') {         // 若指令简写为函数
        dirs[key] = { bind: def, update: def } // 默认在bind和update时执行
      }
    }
  }
}
```


## 背单词

|英文|含义|说明|
|-|-|-|
|crutch|拐杖|不要和裤裆crotch搞混了：open crotch pants开裆裤|
|detrimental|有害的|-|


