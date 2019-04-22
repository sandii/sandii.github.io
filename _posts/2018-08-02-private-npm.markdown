---
layout:     post
title:      "用sinopia搭建私有npm"
subtitle:   "Build Private NPM with Sinopia"
date:       2018-08-02 12:00:00
author:     "Sandii"
header-img: "img/banner/bg-003.jpg"
catalog: true
tags:
    - npm
    - sinopia
---

## 为什么需要私有npm？
因为想方便地管理项目中的公用组件，但又不能把公司的代码随意开源出去，于是在私有服务器上搭建一套npm环境。

## sinopia
先看看官方文档的推荐用法<https://github.com/rlidwka/sinopia>
> 1. Use private packages.
> 1. Cache npmjs.org registry.
> 1. Override public packages.

1. 管理私有包（这就是我们想要的功能）
1. 缓存公有包（这点很关键）
1. 覆盖公有包（也很有用，但暂时还用不上）

其中第二点是关键，我们看一下这一点的详细说明：
> If some package doesn't exist in the storage, server will try to fetch it from npmjs.org...Sinopia will download only what's needed (= requested by clients), and this information will be cached, so if client will ask the same thing second time...

翻译：安装依赖包时，若在本地找不到对应的私有包，会去npmjs.org上找公有包，下载的公有包会被缓存在sinopia上，下次再下载时就不用再去npmjs.org（超级慢）上找了。

这不仅意味着我们在`npm i`时能节约一些时间，还可以让我们一视同仁地安装和更新私有和公有包，非常方便。

开始动手吧。

## 安装
- 确保有nodejs，然后 `npm i -g sinopia`
- 安装后看一下sinopia目录下的内容

|内容|说明|
|-|-|
|config.yaml|配置文件|
|htpasswd|用户名和密码，刚安装时没有，添加用户之后会自动生成|
|storage（目录）|存放包的地方|

## 配置文件 config.yaml
- 启动前要先改改配置文件，下面只摘录关键的选项

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
npm i -g forever                  # 安装forever管理node进程
forever start `which sinopia`     # 启动
forever stop `which sinopia`      # 停止
forever restart `which sinopia`   # 重启
```
- 启动后用浏览器访问`http://服务器地址:4873/`应该就能看到私有npm的页面了
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

nrm add hello http://服务器地址:4873/  # 添加上刚搭起来的私有服务
nrm use hello  # 切过去
nrm test hello # 测试一下能不能连上，能连上的话会显示响应时间

# 试一下 taobao真的比npm快好多
nrm test taobao
nrm test npm
```
- nrm官方文档 <https://github.com/Pana/nrm>


## 用户管理

- 确保客户端连上私有npm服务之后，就可以`npm adduser`注册用户了，和公有npm一样，按提示操作即可。
- 有用户注册以后，在服务端上就能看到`htpasswd`文件了，直接修改这个文件也可以增删用户
- 配置文件中的`packages`项下还有更加详细的权限管理配置，有需要可以研究一下
- 当然不能让所有人都访问我们的私有npm，在项目组的所有同学注册完成之后，修改配置文件，关闭注册通道
```
max_users: -1   # 停止注册
```

## 实操流程

```
# 更新完成后先进行git提交
git add .
git commit -m 'update informations here'
git fetch && git rebase origin/my-branch

# 私有npm发布
npm login         # 第一次使用需要登录，以后就不用了
npm version patch # 打个新的版本号，z++
npm publish       # 发布

# 然后git push
# 这次push包含两次commit
# npm version patch会自动产生一个git commit，提交信息就是本次发布的版本号
git push origin my-branch

# 最后还要去引用了该私有包的项目中安装本次更新
# 安装后会自动更新packages.json中私有包的版本号
cd ../my-project
npm i my-package@latest

# 将更新后的packages.json进行git提交
git add .
git commit -m 'update my-package to v1.0.14'
git fetch && git rebase origin/my-branch
git push origin my-branch
```
- 经过上面的一通操作，组内其他同学在从git上fetch代码时，就会发现`packages.json`有更新，他们需要去确认是否是私有包版本更新，若是，只需要执行一句`npm i`即可安装到最新版本

**全剧终**

## 替代方案
管理私有公共组件还有其他方案，各有利弊，用的时候自己权衡吧：
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
