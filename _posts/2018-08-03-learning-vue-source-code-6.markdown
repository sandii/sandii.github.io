---
layout:     post
title:      "Vue源码学习6：规范化options"
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

进入下一块功能**规范化**，就这三行我们写一篇：

```
normalizeProps(child, vm)
normalizeInject(child, vm)
normalizeDirectives(child)
```