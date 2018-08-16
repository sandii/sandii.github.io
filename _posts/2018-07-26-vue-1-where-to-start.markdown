---
layout:     post
title:      "Vue源码学习1：从何下手?"
subtitle:   "Learning Vue Source Code: Where to Start?"
date:       2018-07-26 12:00:00
author:     "Sandii"
header-img: "img/pexels/001.jpg"
catalog: true
tags:
    - vue
---

> Libraries can also become a crutch if you never venture beyond them ... I’ve emphasized the necessity of learning how things work and not simply what they do. There are many wonderful libraries out there, ... but having no understanding of what’s going on behind the scenes will be detrimental to both you and your web application. -- Jeremy Keith

> 框架就像是拐杖，若你不努力超越它，就会丧失独立行走的能力……我已强调了知其所以然的重要性。现在市面上有很多很棒的框架，但若只是使用它，而不去理解框架背后的原理，对你的项目和你自己都将贻害无穷。 -- Jeremy Keith


## 开始

公家的新项目选型了Vue，每天加班赶进度埋头苦干好几个月，项目终于快上线了。某天看见大神说的上面那段话，十分汗颜，必须学习源码了。然而打开<https://github.com/vuejs/vue>，对于技术和经验都一般的我来说，简直是天书。从哪下手呢？

## 最简模式

既然我水平一般，那就先选择一个最简单的方式。按照[官方教程第一页](https://cn.vuejs.org/v2/guide/installation.html#%E7%9B%B4%E6%8E%A5%E7%94%A8-lt-script-gt-%E5%BC%95%E5%85%A5)中的教导，直接用`script标签`引入库文件，此时`Vue`会被注册为一个全局变量，`new`一下启动应用：

```
<!DOCTYPE html>
<html>
<body>
<div id="app">{{a}}</div>
<script src="https://cdn.jsdelivr.net/npm/vue@2.5.16/dist/vue.js"></script>
<script>
new Vue({
    el : '#app',
    data : { a : 'hello world' },
});
</script>
</body>
<html>
```
复制粘贴成一个html文件，拽到浏览器里。虽然我们这个应用很简陋，但还是运行起来了。


## dist/vue.js

看过教程和文档我们知道，Vue提供的API主要分布于：
1. new Vue()的时候传入各种配置项
1. 调用Vue的静态属性和方法
1. 调用Vue的实例属性和方法

那么我们就可以认为，整套源码主要干了这么的几件事：
1. 声明了一个构造函数Vue
1. 为Vue挂载各种静态属性和方法
1. 为Vue实例挂载各种属性和方法

那么我们就看看`vue.js`是怎么把`Vue`输出为一个全局变量的呢。刚才引入的vue.js位于项目的[dist目录](https://github.com/vuejs/vue/tree/dev/dist)，目录中有好多文件：
```
README.md
vue.common.js
vue.esm.browser.js
vue.esm.js
vue.js
vue.min.js
vue.runtime.common.js
vue.runtime.esm.js
vue.runtime.js
vue.runtime.min.js
```
都是些什么鬼？我水平这么一般，又要晕了……还好作者大神在这里贴心地为大家准备了一个`README.md`，包含一个表格和一堆说明

| | UMD | CommonJS | ES Module |
| --- | --- | --- | --- |
| **Full** | vue.js | vue.common.js | vue.esm.js |
| **Runtime-only** | vue.runtime.js | vue.runtime.common.js | vue.runtime.esm.js |
| **Full (production)** | vue.min.js | | |
| **Runtime-only (production)** | vue.runtime.min.js | | |

现在就明白了：

|关键词|含义|
|-|-|
|runtime|精简版，不含编译器，不能使用template模板|
|common|以commonjs方式输出Vue|
|esm|以es module方式输出Vue|
|min|生产环境用的压缩版，比开发版省略了很多校验|

而`vue.js` `vue.min.js` `vue.runtime.js` `vue.runtime.common.js`这四个都是以umd的方式输出Vue，所以我们标签引入vue.js时，会有全局变量Vue

## build构建
那么这些文件是怎么生成的呢？一般是用`npm run build`。于是我们看看`package.json`，其他的我都看不懂，就看懂了这一行：

```
"build": "node scripts/build.js",
```
`npm run build`命令就是执行`scripts/build.js`这个脚本嘛，毫不犹豫打开它。整个脚本其实就干了两件事：

```
// 1. dist目录不存在的话就创建一个
if (!fs.existsSync('dist')) {
  fs.mkdirSync('dist')
} 

// 2. 执行构建！
build(builds)
```

那么到底在构建些啥呢，我们看看builds是什么

```
let builds = require('./config').getAllBuilds()
```

打开`scripts/config.js`找输出的`getAllBuilds`方法

```
exports.getAllBuilds = () => Object.keys(builds).map(genConfig)
```

这里面又有一个builds变量，找到这个变量，这回水平一般的我也能看懂了，这是构建函数的配置文件，根据它构建出dist目录下的那些文件。我们只看刚才用标签引用的vue.js，其他先不管：

```
const builds = {
    // 省略...
    'web-full-dev': {
        entry: resolve('web/entry-runtime-with-compiler.js'), // 输入
        dest: resolve('dist/vue.js'),  // 输出
        format: 'umd',                 // umd输出
        env: 'development',            // 开发模式
        alias: { he: './entity-decoder' },
        banner,
    },
    // 省略...
}
```

含义是把`web/entry-runtime-with-compiler.js`编译为`dist/vue.js`，以`umd`输出，开发模式。去找`web/entry-runtime-with-compiler.js`就行了。但找了半天又懵逼了，这个项目根本没有`web`这个目录啊…那么web既然不是目录，那就是某个目录的别名吧？果然，在`scripts/alias.js`里找到了别名的定义：

```
web: resolve('src/platforms/web'),
```

所以`dist/vue.js`是由`src/platforms/web/entry-runtime-with-compiler.js`编译来的。而这个文件名的含义也证实了我们的猜测：它是精简版+编译器的入口。打开它，最后一行果然是`export default Vue`。


## 追根溯源
打开`src/platforms/web/entry-runtime-with-compiler.js`，有一句这个`import Vue from './runtime/index'`这说明，变量`Vue`也是它从别处引用来的，那么我们就追根溯源：

```
// src/platforms/web/entry-runtime-with-compiler.js
import Vue from './runtime/index'

// src/platforms/web/runtime/index.js
import Vue from 'core/index'

// src/core/index.js
import Vue from './instance/index'

// src/core/instance/index.js
function Vue (options) {
    // ...
}
// 终于找到了！
```

终于在`src/core/instance/index.js`里找到了Vue的真实面目，一个简单的构造函数：
```
function Vue (options) {
  if (process.env.NODE_ENV !== 'production' &&
    !(this instanceof Vue)
  ) {
    warn('Vue is a constructor and should be called with the `new` keyword')
  }
  this._init(options)
}
```


## 开发环境中的代码校验

`if (process.env.NODE_ENV !== 'production)'`这句话从语义上我们就可以知道是用来判断是`开发环境`还是`生产环境`，如果是开发环境的话会判断`if (this instanceof Vue)`即函数Vue是否是用new调用的。若不是的话会发出警告。这说明，Vue是一个构造函数，必须使用new调用，而不能作为普通函数调用。

此类的判断在整套源码里会有很多：
- 开发环境中，严格校验，确保代码正确
- 生产环境中，不再校验，提高代码效率

那么`process.env.NODE_ENV`是在哪里设置上的呢？这还得回到上一篇文章提到过的构建流程，构建配置文件`scripts/config.js`：

```
const builds = {
    // 省略...
    'web-full-dev': {
        entry: resolve('web/entry-runtime-with-compiler.js'),
        dest: resolve('dist/vue.js'),
        format: 'umd',
        env: 'development',     // 开发环境
        alias: { he: './entity-decoder' },
        banner
    },
    'web-full-prod': {
        entry: resolve('web/entry-runtime-with-compiler.js'),
        dest: resolve('dist/vue.min.js'),
        format: 'umd',
        env: 'production',      // 生产环境
        alias: { he: './entity-decoder' },
        banner
    },
    // 省略...
}
```

所以：
- vue.js 开发环境 代码校验
- vue.min.js 生产环境 不做校验


## 源码结构

忽略了开发环境中的代码校验之后，代码就剩下了这么点儿了，这就是整套源码的源头：

```
function Vue (options) {
  this._init(options)
}
```

看到这里，整套源码的结构也就已经很清楚了：

```
// 1. 声明构造函数Vue，执行_init方法
function Vue (options) {
  this._init(options)
}

// 2. 给构造函数Vue挂载静态属性/方法和实例属性/方法
// 这也正是下一步需要去依次探索的地方
// src/core/instance/index.js
// src/core/index.js
// src/platforms/web/runtime/index.js
// src/platforms/web/entry-runtime-with-compiler.js

// 3. 输出Vue
export default Vue

// 4. 把输出的Vue打包成各种版本放入dist目录
// scripts/build.js
```

## 尾声

尽管我水平一般，一行一行看过去，可能很多代码细节暂时根本看不懂。但是我们可以发现这其实这并不影响我们对代码整体的理解。估计在探索新世界的过程中，跳过暂时看不懂的部分也将是常态。


## 背单词

|英文|含义|说明|
|-|-|-|
|crutch|拐杖|不要和裤裆crotch搞混了：open crotch pants开裆裤|
|detrimental|有害的|-|
