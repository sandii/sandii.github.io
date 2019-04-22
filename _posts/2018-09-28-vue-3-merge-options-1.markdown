---
layout:     post
title:      "Vue源码学习3：详解mergeOptions函数（上）"
subtitle:   "Learning Vue Source Code 3: MergeOptions Function Insight 1"
date:       2018-09-15 12:00:00
author:     "Sandii"
header-img: "img/pexels/003.jpg"
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

虽然每个词都能看懂，但还是不明白是什么意思，汗。


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

这句大概的意思是，用`mergeOptions`函数处理用户传入的`options`对象，把处理结果赋值给Vue实例对象的`$options`属性。`$options`属性在API中可以查到，是开放给用户使用的。

`mergeOptions`函数是从外部引入的：

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

既然是用于处理`options`的代码，那么打开`src/core/util/options.js`，猜对了~
但是400多行的代码，加上各种外部引入的东西，估计够我看一阵子了。

直奔主题，`mergeOptions`函数的大致结构是这样的：

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

跳过子类的逻辑后，我们可以认为这个函数只有一句`return Vue.options`。这说明Vue上本身就挂载了一个options属性，在文档上查不到，说明也是源码内部使用的。它是什么时候被挂载上来的我们还不知道，但是却没法跳过它，因为如果不知道**parent是什么**的话我们就完全没法理解后面的合并操作。

只能先作个弊，回到我们那个简陋的应用，在浏览器控制台里输入一句`Vue.options`:

![](img/content/001.jpg)

这说明`Vue.options`定义了Vue内置功能，比如`component`里有`transition`就意味着我们在模板里可以随时使用`transition`标签；`directive`里有`model`和`show`说明我们可以随时使用`v-show`和`v-model`指令。

所以，`mergeOptions`的功能是合并 **用户传入的options** 和 **Vue.options**，作为真正实例化时的配置。


## 子组件命名的三个条件

终于进入mergeOptions函数的内部了。

```
// 若是开发环境，检查用户传入的options中的components这一项
if (process.env.NODE_ENV !== 'production') {
  checkComponents(child)
}
// 校验每个子组件名
function checkComponents (options: Object) {
  for (const key in options.components) {
    validateComponentName(key)
  }
}
// 若自组件名不合符以下三个条件就发出警告
export function validateComponentName (name: string) {
  if (!/^[a-zA-Z][\w-]*$/.test(name)) {
    warn(
      'Invalid component name: "' + name + '". Component names ' +
      'can only contain alphanumeric characters and the hyphen, ' +
      'and must start with a letter.'
    )
  }
  if (isBuiltInTag(name) || config.isReservedTag(name)) {
    warn(
      'Do not use built-in or reserved HTML elements as component ' +
      'id: ' + name
    )
  }
}
```

我们知道components中的key就是在template中所使用的自定义标签，检验自定义标签的命名规范是很有必要的。以上代码逻辑比较简单：开发环境时检查子组件命名是否符合规范，不符合规范发出警告。组件命名规范有3条：

1. `/^[a-zA-Z][\w-]*$/` 必须以字母开头，除了连字符`-`之外不能有其他符号
1. `isBuiltInTag` 不能和Vue的内建组件重名
1. `config.isReservedTag` 不能和系统保留标签重名

`isBuiltInTag函数`很好找，在`src/shared/util.js`：

```
// 虽然不知道makeMap函数是什么意思，但还是能猜出
// 子组件不能叫slot或component
export const isBuiltInTag = makeMap('slot,component', true)
```


## 寻找config.isReservedTag

`config.isReservedTag`就没那么好找了。我们在`src/core/config`里只找到了这么一句:

```
isReservedTag: no,
```

`no`在`src/shared/util.js`，是一个永远返回`false`的函数：

```
// no 永远返回false
export const no = (a?: any, b?: any, c?: any) => false

// 类似的还有noop和identity：
// noop 啥也不做的函数，估计是no operation的错写
export function noop (a?: any, b?: any, c?: any) {}

// identity 传啥返啥的函数
export const identity = (_: any) => _
```

以上代码说明，不论子组件叫什么名字都可以，这显然是不符合逻辑的。


## 执行环境相关功能

但更有价值的是这一段注释：

> Check if a tag is reserved so that it cannot be registered as a component. This is platform-dependent and may be overwritten.

> 翻译：检验是否是保留标签，若是保留字是不能注册为组件的。而这个特性根据不同执行环境会有所不同，因此可能会在别处被重写。

所以说，`config.isReservedTag`和许多其他特性是根据不同的执行环境在**某处**，被挂载上的。到底是在哪里呢？在之前顺藤摸瓜寻找Vue的源头的过程中，我们已经知道，Vue从被声明到输出，一路走来有着丰富的经历（怎么感觉怪怪的）：

```
src/platforms/web/entry-runtime-with-compiler.js
src/platforms/web/runtime/index.js
src/core/index.js
src/core/instance/index.js
```

看到这里，`platforms/web`这个路径的含义也就明白了：添加web环境相关的功能。除了web还有其他环境吗？有啊，`platforms`目录下除了web还有weex。总之，基本上可以肯定`config.isReservedTag`就是在这俩里面被添加上的：

```
src/platforms/web/entry-runtime-with-compiler.js
src/platforms/web/runtime/index.js
```

果然在`src/platforms/web/runtime/index.js`找到了：

```
import { isReservedTag } from 'web/util/index'
Vue.config.isReservedTag = isReservedTag

// src/platforms/web/util/index.js 又是一个大汇总
// 我们转到 src/platforms/web/util/element.js 才最终找到
// 它汇总了html和svg标签：
export const isReservedTag = (tag: string): ?boolean => {
  return isHTMLTag(tag) || isSVG(tag)
}
export const isHTMLTag = makeMap(
  'html,body,base,head,link,meta,style,title,' +
  'address,article,aside,footer,header,h1,h2,h3,h4,h5,h6,hgroup,nav,section,' +
  'div,dd,dl,dt,figcaption,figure,picture,hr,img,li,main,ol,p,pre,ul,' +
  'a,b,abbr,bdi,bdo,br,cite,code,data,dfn,em,i,kbd,mark,q,rp,rt,rtc,ruby,' +
  's,samp,small,span,strong,sub,sup,time,u,var,wbr,area,audio,map,track,video,' +
  'embed,object,param,source,canvas,script,noscript,del,ins,' +
  'caption,col,colgroup,table,thead,tbody,td,th,tr,' +
  'button,datalist,fieldset,form,input,label,legend,meter,optgroup,option,' +
  'output,progress,select,textarea,' +
  'details,dialog,menu,menuitem,summary,' +
  'content,element,shadow,template,blockquote,iframe,tfoot'
)
// this map is intentionally selective, only covering SVG elements that may
// contain child elements.
export const isSVG = makeMap(
  'svg,animate,circle,clippath,cursor,defs,desc,ellipse,filter,font-face,' +
  'foreignObject,g,glyph,image,line,marker,mask,missing-glyph,path,pattern,' +
  'polygon,polyline,rect,switch,symbol,text,textpath,tspan,use,view',
  true
)
```

然后还要看一下`makeMap函数`。

```
/**
 * Make a map and return a function for checking if a key
 * is in that map.
 */
export function makeMap (
  str: string,
  expectsLowerCase?: boolean
): (key: string) => true | void {
  const map = Object.create(null)
  const list: Array<string> = str.split(',')
  for (let i = 0; i < list.length; i++) {
    map[list[i]] = true
  }
  return expectsLowerCase
    ? val => map[val.toLowerCase()]
    : val => map[val]
}
```

代码清晰易懂：
- 生成一个map，返回一个函数，看key是否在map中
- 将map储存在闭包中，每次访问时间复杂度`O(1)`，而直接使用数组`O(n)`


## 标准化 props inject directives

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


那么为什么要标准化呢？官方文档告诉我们`props` `inject` `directive`这三个选项可兼容多种不同的写法。标准化就是把这多种写法统一成一种，便于后面处理。比如`props`可以有三种写法:

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
  if (Array.isArray(props)) { // 1. 处理数组
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
  options.props = res // 覆盖原有的props
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


## 总结一下

- `mergeOptions`函数的功能是合并`用户传入的options`和`Vue.options`，赋给`vm.$options`，作为真正实例化时的配置。
- 合并options一开始就是校验子组件名，校验的3个规则是：
1. 字母开头，不能有连字符之外的符号
1. 不能是vue内置标签
1. 不能是html和svg标签
- 标准化 `props` `inject` `directive`，因为他们可以有多种写法

- Vue的运行环境可能有`web`和`weex`，源码中有许多功能是根据不同的环境添加的。


## 背单词

|英文|类型|含义|
|-|-|-|
|hyphen|连字符|
|delimit|界定 分隔|
|instance|noun|实例|
|instantiate|verb|实例化|
|instantiation|noun|实例化|

注意区分`instant`瞬间
