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
博客停更了将近4个月。对我来说，这4个月仿佛4年那么长。

去年秋天我们开始动手换房。为了孩子上一个稍好一点的学校，我们贷了更多的款，借了更多的钱，经过了好几场谈判，和买家卖家勾心斗角终于置换了房子。但事情办到一半，妈妈被确诊癌症，然后：联系医院、准备红包、手术、陪床、买菜做饭、化疗、保肝、复查……现在情况终于暂时稳定下来，房子也装修完通风了4个月可以入住了。我经历了需要经历的一切。哭过，笑过，更多地是咬紧牙关向前冲。

几点感受：

1. 无论什么时候，健康都是最重要的。健康是一切的前提，所以为了自己和家人的健康，我们应当有所取舍。经历过了这一切，我觉得就能从一个更加理性和客观的角度看待最近热议的996.icu。
1. 检验一个人经济状况的标准不是买了几套房，而是进了医院心里慌不慌。
1. 人的潜力是无限的。半个月之内，我的体重从140斤降到120斤，现在经过饮食和锻炼很快又回到140斤。
1. 医学的进步比想象中的还要慢，和20年前相比并没有本质的进展。
1. 造物者到底为什么要设计癌症这个特性？！是对人类的惩罚吗？
1. 但尽人事，等待命运降临。

总之，终于可以暂时回到日常的工作和学习了，重新拿起键盘写代码的感觉真的很幸福。从这个角度上来说，马云爸爸说的“996是一种幸福”就可以理解了吧。


## 导语
忘了是什么时候，重新安装nodejs后，项目里出现了一个`package-lock.json`文件，尺寸还不小，好几百k。每次执行`npm i`后都会有所修改。

可能是我的搜索方式不对，上网查了一圈，都没有一个让人满意的解释。于是打开全知全能的stackoverflow。问题迎刃而解，屡试不爽，清清楚楚…

摘录其中个人觉得最好的一部分，翻译出来和大家分享：


## 提问

新发布的npm@5包含了一个新特性：执行npm install之后产生了一个package-lock.json文件。想问问这个文件是干什么用的？


## 回答

`package-lock.json`记录了项目依赖树中所有依赖包的具体版本。而`package.json`中常常使用通配符来描述依赖包的版本（比如1.0.*）。这可以保证开发团队中所有开发者以及项目发布的生产环境代码所使用的依赖包版本完全一致。


It stores an exact, versioned dependency tree rather than using starred versioning like package.json itself (e.g. 1.0.*). This means you can guarantee the dependencies for other developers or prod releases, etc. It also has a mechanism to lock the tree but generally will regenerate if package.json changes.

From the npm github pages:

package-lock.json is automatically generated for any operations where npm modifies either the node_modules tree, or package.json. It describes the exact tree that was generated, such that subsequent installs are able to generate identical trees, regardless of intermediate dependency updates.

This file is intended to be committed into source repositories, and serves various purposes:

Describe a single representation of a dependency tree such that teammates, deployments, and continuous integration are guaranteed to install exactly the same dependencies.

Provide a facility for users to "time-travel" to previous states of node_modules without having to commit the directory itself.

To facilitate greater visibility of tree changes through readable source control diffs.

And optimize the installation process by allowing npm to skip repeated metadata resolutions for previously-installed packages."

## 追问

If having an exact version of dependencies is so sought after, why not enforce specifying the exact version in package.json and forgoe a package-lock.json file?

## 回答追问

To answer jrahhali's question below about just using the package.json with exact version numbers. Bear in mind that your package.json contains only your direct dependencies, not the dependencies of your dependencies (sometimes called nested dependencies). This means with the standard package.json you can't control the versions of those nested dependencies, referencing them directly or as peer dependencies won't help as you also don't control the version tolerance that your direct dependencies define for these nested dependencies.

Even if you lock down the versions of your direct dependencies you cannot 100% guarantee that your full dependency tree will be identical every time. Secondly you might want to allow non-breaking changes (based on semantic versioning) of your direct dependencies which gives you even less control of nested dependencies plus you again can't guarantee that your direct dependencies won't at some point break semantic versioning rules themselves.

The solution to all this is the lock file which as described above locks in the versions of the full dependency tree. This allows you to guarantee your dependency tree for other developers or for releases whilst still allowing testing of new dependency versions (direct or indirect) using your standard package.json.

NB. The previous shrink wrap json did pretty much the same thing but the lock file renames it so that it's function is clearer. If there's already a shrink wrap file in the project then this will be used instead of any lock file.

## 总结


## 背单词

|英文|翻译|
|-|-|-|
|||
|||
|||
|||

## 原文
<https://stackoverflow.com/questions/44297803/package-lock-json-role>
