---
layout:     post
title:      "Vue源码学习2：构造函数"
subtitle:   "Learning Vue Source Code 2： Constructor"
date:       2018-07-21 12:00:00
author:     "Sandii"
header-img: "img/banner/bg-006.jpg"
catalog: true
tags:
    - vue
---


## 书接上回

看过教程和文档我们知道，Vue提供的API主要分布于：
1. new Vue()的时候传入各种配置项
1. 调用Vue的静态属性和方法
1. 调用Vue的实例属性和方法

那么我们就可以认为，整套源码主要干了这么的几件事：
1. 声明了一个构造函数Vue
1. 为Vue挂载各种静态属性和方法
1. 为Vue实例挂载各种属性和方法

上一篇文章里我们已经找到了新世界的入口`src/platforms/web/entry-runtime-with-compiler.js`，那么我们就看一下，它是怎么一步一步增加上API中的那些功能的。


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
		env: 'development', 	// 开发环境
		alias: { he: './entity-decoder' },
		banner
	},
	'web-full-prod': {
		entry: resolve('web/entry-runtime-with-compiler.js'),
		dest: resolve('dist/vue.min.js'),
		format: 'umd',
		env: 'production',		// 生产环境
		alias: { he: './entity-decoder' },
		banner
	},
	// 省略...
}
```

所以：
- vue.js 开发环境 代码校验
- vue.min.js 生产环境 不做校验


## 总结

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
// 这也正是下一步需要去看的地方
// src/core/instance/index.js
// src/core/index.js
// src/platforms/web/runtime/index.js
// src/platforms/web/entry-runtime-with-compiler.js

// 3. 输出Vue
export default Vue

// 4. 把输出的Vue打包成各种版本放入dist目录
// scripts/build.js
```


> 封面图： 槐柏树街 - 2013春 - Sandii
