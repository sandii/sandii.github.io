---
layout:     post
title:      "Vue源码学习10： renderProxy"
subtitle:   "Learning Vue Source Code 10: renderProxy"
date:       2018-08-08 12:00:00
author:     "Sandii"
header-img: "img/pexels/010.jpg"
catalog: true 
tags:
    - vue
---

费了好大劲终于把`mergeOptions()`看完了。我们继续读`Vue.prototype._init()`方法。

```
if (process.env.NODE_ENV !== 'production') {
  initProxy(vm)
} else {
  vm._renderProxy = vm
}
```

打开`src/core/instance/proxy.js`，整个文件不过几十行，这篇就专门写这几十行了。

```

```