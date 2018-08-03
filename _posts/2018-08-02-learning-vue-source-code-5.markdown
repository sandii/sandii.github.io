---
layout:     post
title:      "Vue源码学习5：检查组件名"
subtitle:   "Learning Vue Source Code 5: check component name"
date:       2018-08-02 12:00:00
author:     "Sandii"
header-img: "img/pexels/005.jpg"
catalog: true
tags:
    - vue
---

终于进入mergeOptions函数的内部了。一上来先校验child。

## 子组件命名的三个条件

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

// isBuiltInTag方法在src/shared/util.js
export const isBuiltInTag = makeMap('slot,component', true)

// config.isReservedTag
config.isReservedTag
```

以上代码逻辑比较简单：开发环境时检查子组件命名是否符合规范，不符合规范发出警告。组件命名规范有3条：

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


## 总结一下

合并options一开始就是校验子组件名，校验的3个规则是：
1. 字母开头，不能有连字符之外的符号
1. 不能是vue内置标签
1. 不能是html和svg标签

Vue的运行环境可能有`web`和`weex`，源码中有许多功能是根据不同的环境添加的。

## makeMap函数

哦，最后还要看一下`makeMap函数`。

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

- 生成一个map，返回一个函数，看key是否在map中
- 将map储存在闭包中，每次访问时间复杂度`O(1)`，而直接使用数组`O(n)`

