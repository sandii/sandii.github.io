---
layout:     post
title:      "为什么 UNIX/Linux 不允许目录硬链 【翻译】"
subtitle:   "Why Are Hard Links to Directories Not Allowed in UNIX/Linux?"
date:       2018-10-30 12:00:00
author:     "Sandii"
header-img: "img/banner/bg-004.jpg"
catalog: true
tags:
    - linux
    - filesystem
---

## 导语：
在学习Linux文件系统的硬链接和符号连接时，我也产生了这个疑问。搜索找到几篇中文资料，总感觉像是隔靴搔痒，似懂非懂。改用英文搜索，找到了这篇文章，短短几句话，一针见血，深入浅出。于是翻译出来与大家分享。

## 提问：

教科书中说，Unix/Linux不允许硬链接指向目录，但软链接却可以。这是因为目录的硬链接会在文件系统中产生循环吗？还是因为如果我们给目录创建了硬链接后再删除原目录，会导致硬链接指向垃圾数据呢？若只是因为文件系统循环就禁用目录硬链接的话，那为什么又允许使用目录的软链接呢？


## 回答：

硬链接指向目录是行不通的。因为我们没有办法区分目录和它的硬链接，两者没有任何区别。

如果允许目录的硬链接，就有可能产生循环目录或空挂的子目录树，从而破坏文件系统的`有向无环图`（directed acyclic graph）结构，这将使`fsck`等用于遍历文件树的命令无法运行。

要理解这一点，我们首先来讨论一下`inode`。文件系统中的数据储存在磁盘的`block`中，每个inode管理着若干个block。我们可以认为这个inode就是`文件`。但inode无法记录文件名。这时我们需要`link`来帮忙了。

`link`是一个指向inode的指针，同时记录了文件名。`目录`也是一个inode，它的blocks保存的数据是若干link。

硬链接也指向inode。观察`ls -l`的输出，权限字段后面的那个数字的含义是指向该inode的链接数。大多数普通文件只有一个链接。而创建一个硬链接会使两个链接指向同一个inode。

```
% ls -l test
ls: test: No such file or directory
% touch test
% ls -l test
-rw-r--r--  1 danny  staff  0 Oct 13 17:58 test
% ln test test2
% ls -l test*
-rw-r--r--  2 danny  staff  0 Oct 13 17:58 test
-rw-r--r--  2 danny  staff  0 Oct 13 17:58 test2
% touch test3
% ls -l test*
-rw-r--r--  2 danny  staff  0 Oct 13 17:58 test
-rw-r--r--  2 danny  staff  0 Oct 13 17:58 test2
-rw-r--r--  1 danny  staff  0 Oct 13 17:59 test3
            ^
            ^ this is the link count
```

现在，我们可以清楚的发现，其实根本没有`硬链接`这个东西。所谓的硬链接其实就是一个`link`。在上面的例子中，我们根本没法分辨出test和test2哪个是原文件那个是硬链接（连时间戳都一样）。因为，这两个文件名指向同样的inode，指向同样的数据。

```
% ls -li test*  
14445750 -rw-r--r--  2 danny  staff  0 Oct 13 17:58 test
14445750 -rw-r--r--  2 danny  staff  0 Oct 13 17:58 test2
14445892 -rw-r--r--  1 danny  staff  0 Oct 13 17:59 test3
```

参数`-i`用于显示文件的inode号。我们可以看到test和test2的inode号相同，而test3不同。

好了，如果我们为目录创建一个硬链接，那么位于文件系统中不同位置的两个目录就会指向同一内容。我们可以让子目录指向祖父目录，从而产生一个循环。

为什么不能有循环呢？因为遍历文件树时，我们没法知道我们是不是在兜圈子（除非一边遍历一边记录inode号）。比如，`du`命令会递归地遍历所有子目录来计算磁盘使用量。而循环会让`du`陷入麻烦，它不得不做大量的记录工作来完成原本简单的任务。

而`符号链接`（Symlinks）则是完全不同的一个物种了。它是一类特殊的文件类型，很多文件系统都支持符号链接。（win系统中的`快捷方式`就是符号链接————译者注）。注意，符号链接可以指向一个不存在的文件，因为它指向文件名而非inode。硬链接则不能指向空文件，只要硬链接存在就意味着文件存在。

那么为什么`du`能轻松处理目录的符号链接却不支持目录的硬链接呢？经过上面的讨论我们知道，目录的硬链接和普通目录是同一个东西。而符号链接则是一种特殊的文件类型，`du`能够检测出来并在遍历文件树时跳过它！

```
% ls -l 
total 4
drwxr-xr-x  3 danny  staff  102 Oct 13 18:14 test1/
lrwxr-xr-x  1 danny  staff    5 Oct 13 18:13 test2@ -> test1
% du -ah
242M    ./test1/bigfile
242M    ./test1
4.0K    ./test2
242M    .
```
By Danny Dulai & G-Man

## 背单词

|英文|含义|
|-|-|
|directed acyclic graph|有向无环图|
|symlink / symbol link|符号链接/软链接|


## 原文
<https://unix.stackexchange.com/questions/22394/why-are-hard-links-to-directories-not-allowed-in-unix-linux?from=timeline>


> 封面图： 天山 - 2015夏 - Sandii
