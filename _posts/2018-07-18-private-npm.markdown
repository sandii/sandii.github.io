---
layout:     post
title:      "用sinopia搭建私有npm"
subtitle:   "build private npm with sinopia"
date:       2018-07-19 12:00:00
author:     "Sandii"
header-img: "img/banner/bg-003.jpg"
catalog: true
tags:
    - npm
    - sinopia
---

## 为什么需要私有npm？
答：想方便地管理项目中的代码包，但又不能把公司的代码随意开源出去，于是我们在私有服务器上搭建一套npm环境

## sinopia
先看看官方文档的推荐用法<https://github.com/rlidwka/sinopia>
> 1. Use private packages.
> 1. Cache npmjs.org registry.
> 1. Override public packages.

1. 管理私有包（这就是我们想要的功能）
1. 缓存公有包（这点很关键）
1. 覆盖公有包（也很有用，但我们暂时还用不上）

其中第二点是关键，我们看一下这一点的详细说明：
> If some package doesn't exist in the storage, server will try to fetch it from npmjs.org...Sinopia will download only what's needed (= requested by clients), and this information will be cached, so if client will ask the same thing second time...

翻译：安装依赖包时，若在本地找不到对应的私有包，会去npmjs.org上找公有包，下载的公有包会被缓存在sinopia上，下次再下载时就不用再去npmjs.org（超级慢）上找了。

这不仅意味着我们在`npm i`时能节约一些时间，还可以一视同仁地安装和更新私有和公有包，真方便。

开始动手吧。

## 安装
- 确保有nodejs，然后 `npm i -g sinopia`

## 配置文件 config.yaml
- 先看一下sinopia的目录下的三个东西

|内容|说明|
|-|-|
|config.yaml|配置文件|
|htpasswd|用户名和密码，安装时没有，添加用户之后会自动生成|
|storage（目录）|存放包的地方|

- config.yaml（只摘录重要的选项）

```
storage: ./storage  # 存放代码包的目录

auth:
  htpasswd:
    file: ./htpasswd # 保存账号密码的文件
    max_users: 1000  # 改为-1可以禁止用户注册

uplinks:
  npmjs:
    url: http://registry.npm.taobao.org/  # 下载公有包的地址

# 关键是这句，不写这句的话，只能本机访问，外网是访问不了的
listen: 0.0.0.0:4873  
```
## 启动
```
// 安装forever管理node进程
$ npm i -g forever

// 启动
$ forever start `which sinopia`
```
- 这时用浏览器访问`http://服务器地址:4873/`应该就能看到私有npm的页面了
- 这样服务端就算搞定了

## 客户端配置
- 首先需要把客户端npm的registry指向我们刚搭起来的私有服务

```
npm set registry http://服务器地址:4873/
```
- 但这么搞太麻烦了，我们安装一个npm源管理工具`nrm` (npm registry manager)

```
npm i -g nrm   # 安装nrm
nrm ls         # 看看有什么可用的源
nrm use taobao # 平时可以直接用淘宝的镜像

# 不玩了 干正事..
nrm add hello http://服务器地址:4873/  # 添加上刚搭起来的私有服务
nrm use hello  # 切过去
nrm test hello # 测试一下能不能连上，能连上的话会显示响应时间

# 试一下 taobao真的比npm快好多
nrm test taobao
nrm test npm
```
- nrm官方文档 <https://github.com/Pana/nrm>


## 用户管理
- 确保客户端能连上服务器后，就可以注册用户了
```
npm adduser  # 和公有npm一样，按提示操作
```
- 当然不能让所有人都访问我们的私有npm，在所有用户注册完成之后，要记得修改配置文件，关闭注册通道
```
max_users: -1
```
- 还可以手动修改`htpasswd`文件来管理用户
- 配置文件中的`packages`项下还有更加详细的权限管理配置，有需要可以研究一下


## 实操流程

1. 更新发布。和公有包没啥区别，只需要确认registry指向自己的私有服务即可。

```
npm login         # 登录
npm version patch # 打个新的版本号，z++
npm publish       # 发布
# 然后是git发布，省略..
```

2. 私有包的更新者需要到引用该私有包的所有项目中进行更新。

```
npm i package-name@latest # 这样会修改packages.json中私有包的版本号
# 把修改过的packages.json发布到git上，省略..
```

3. 其他开发者在从git上fetch代码时，若发现`packages.json`有更新，就需要确认是否是私有包版本更新，若是，`npm i`即可

**全剧终**

## 替代方案
当然还有其他替代的方案，各有利弊，用的时候自己权衡吧：
1. git in package.json：简单易用，但不能用`npm update`，更新不太方便
1. cnpm：要安装数据库，好麻烦


## 背单词

|英文|含义|
|-|-|
|npm|nodejs packages manager|
|nrm|npm registry manager|

## 参考

- <https://pines-cheng.github.io/blog/#/posts/1> pines cheng
- <https://github.com/rlidwka/sinopia> rlidwka 
- <https://github.com/Pana/nrm> pana

> 封面图： 天坛公园 - 2014夏 - Sandii
