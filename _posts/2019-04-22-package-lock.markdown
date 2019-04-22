---
layout:     post
title:      "package-lock.json到底是做什么用的 【编译】"
subtitle:   "what on earth the use of package-lock.json"
date:       2018-12-12 12:00:00
author:     "Sandii"
header-img: "img/pexels/009.jpg"
catalog: true 
tags:
  - nodejs
  - npm
---


## 题外话

博客停更了将近好久。这真是一段艰难的日子。

去年秋天我们开始动手换房。为了孩子上一个稍好一点的学校，我们贷了更多的款，借了更多的钱，经过了几场和买家卖家勾心斗角的谈判，终于置换了房子。但事情办到一半，妈妈被确诊癌症，然后：联系医院、准备红包、手术、陪床、买菜做饭、化疗、保肝、复查……现在情况终于暂时稳定下来，房子也装修完通风了4个多月可以入住了。我经历了需要想象得到和想象不到的各种情形，哭过，笑过，更多地是咬紧牙关向前冲。

几点感受：

1. 无论什么时候和什么情况下，健康都是最重要的。经历过了这一切，我觉得就能从一个更加理性和客观的角度看待最近热议的996.icu。健康是一切的前提，所以为了自己和家人的健康，我们应当有所取舍。
1. 检验一个人经济状况的标准不是买了几套房，而是进了医院心里慌不慌。
1. 人的潜力是无限的。半个月之内，我的体重从140斤降到120斤，现在经过饮食和锻炼很快又回到140斤。
1. 医学的进步比想象中的还要慢，和20年前相比并没有本质的进展。
1. 造物者到底为什么要设计癌症这个特性？！是对人类的惩罚吗？
1. 但尽人事，等待命运降临。
1. 保持乐观。

总之，终于可以暂时回到日常的工作和学习了，重新拿起键盘写代码的感觉真的很幸福。从这个角度上来说，马云爸爸说的“996是一种幸福”就可以理解了吧。


## 导语

忘了是什么时候开始，重新安装nodejs后，项目里出现了一个`package-lock.json`文件，尺寸还不小，好几百k。每次执行`npm i`后都会修改，挺烦人的。

上网查了一圈，都没有一个让看明白的解释，可能是我的搜索方式不对？于是打开全知全能的`stackoverflow`。问题迎刃而解，屡试不爽，清清楚楚…

摘录其中个人觉得说的很好的部分，翻译出来和大家分享：


## 提问

新发布的`npm@5`包含了一个新特性：执行`npm install`之后产生了一个`package-lock.json`文件。想问问这个文件是干什么用的？


## 回答

`package-lock.json`记录了项目依赖树中所有依赖包的具体版本。而`package.json`中常常使用通配符来描述依赖包的版本（比如1.0.*）。这可以保证团队中所有开发者以及项目发布的生产环境代码所使用的依赖包版本完全一致。而且若`package.json`发生了变化，`package-lock.json`中的依赖树也会进行相应的更新。


## 追问

若只是为了记录确保依赖包的精确版本，为何不在`package.json`中就精确指定，而非要使用一个`package-lock.json`文件呢？


## 回答追问

因为`package.json`只包含了直接依赖，不包括依赖的依赖（嵌套依赖）。这意味着`package.json`不能控制嵌套依赖包的版本。就算你把这些嵌套依赖直接写进`package.json`的`depencies`字段也不行，因为项目中的直接依赖包会在它们的`package.json`中定义嵌套依赖的版本。

所以目前针对这个问题的解决方案就是增加一个专门的文件，锁定项目完整的依赖树版本。这样我们既可以保证整个开发团队和发布的生产环境代码的依赖树的一致性，也可以通过调整`package.json`来测试不同的依赖包版本。

注意，之前的`shrinkwrap.json`和现在`package-lock.json`的功能是一样的，改名后它的功能更清楚了。如果项目中已经有`shrinkwrap.json`文件，那么npm会优先读取它。


## 官方文档

回答者同时还引用了npm官方文档对`package-lock.json`的说明：

只要`npm`修改`node_modules树`或`package.json`，就会自动生成一份`package-lock.json`文件，这个文件精确描述了项目的整个`node_modules树`。在此之后的`npm install`命令将按照`package-lock.json`文件的内容生成`node_modules树`。

我们建议将`package-lock.json`提交到项目的源码仓库中，它有以下几个作用：

- 保证所有开发者的开发版本和部署版本的依赖包完全相同；
- 在不用提交node_modules目录本身的情况下，允许用户回滚到node_modules的任意之前版本；
- 使依赖树的变化可视化；
- 优化npm install：跳过已安装过的包


## 总结

记得`Ryan Dahl`谈为什么要放弃node开发deno时提到他认为nodejs目前的一个严重的问题是`node_modules`实在太深了。

![](https://pics.onsizzle.com/sun-neutron-star-black-hole-node-modules-heaviest-objects-in-the-27126154.png)

专门开辟一个`package-lock.json`来管理`node_modules`也就可以理解了。


## 背单词

这三个都是拉丁文，但实在太常用了，必须要会读和会背。

|英文|拉丁文|翻译|
|-|-|-|
|e.g.|exempli gratia|例如|
|etc|et cetera|等等|
|NB|nota bene|注意|


## 原文
<https://stackoverflow.com/questions/44297803/package-lock-json-role>
