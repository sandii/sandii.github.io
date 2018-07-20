---
layout:     post
title:      "为什么Unix/Linux不允许硬链接指向目录 【翻译】"
subtitle:   "Why Are Hard Links to Directories Not Allowed in UNIX/Linux?"
date:       2018-07-21 12:00:00
author:     "Danny Dulai & G-Man"
header-img: "img/banner/bg-004.jpg"
catalog: true
tags:
    - linux
---

## 导语：
在学习Linux文件系统的硬链接和软连接时，我也产生了这个疑问。搜索一番，阅读了几篇中文资料，总感觉像是隔靴搔痒，似懂非懂。改用英文搜索，找到了这篇文章，短短几句话，深入浅出，一针见血，让我恍然大悟。于是翻译出来与大家分享。 By Sandii.

## 提问：

I read in text books that Unix/Linux doesn't allow hard links to directories but does allow soft links. Is it because, when we have cycles and if we create hard links, and after some time we delete the original file, it will point to some garbage value?

教科书中说，Unix/Linux不允许硬链接指向目录，但软链接却可以。这是因为目录的硬链接会在文件系统中产生循环吗？还是因为如果我们给目录创建了硬链接后再删除原目录，会导致硬链接指向垃圾数据呢？


If cycles were the sole reason behind not allowing hard links, then why are soft links to directories allowed?

若只是因为文件系统循环就禁用目录硬链接的话，那为什么又允许使用目录的软链接呢？


## 回答：

This is just a bad idea, as there is no way to tell the difference between a hard link and an original name.

硬链接指向目录是行不通的。因为我们没有办法区分目录和它的硬链接，两者没有任何区别。


Allowing hard links to directories would break the directed acyclic graph structure of the filesystem, possibly creating directory loops and dangling directory subtrees, which would make fsck and any other file tree walkers error prone.

如果允许目录的硬链接，就有可能产生循环目录或空挂的子目录树，从而破坏文件系统的`有向非循环图`结构(directed acyclic graph structure)，这将使`fsck`等用于遍历文件树的命令无法运行。


First, to understand this, let's talk about inodes. The data in the filesystem is held in blocks on the disk, and those blocks are collected together by an inode. You can think of the inode as THE file.  Inodes lack filenames, though. That's where links come in.

要理解这一点，我们首先来讨论一下`inode`。文件系统中的数据储存在磁盘的`block`中，每一群`block`都有一个`inode`来组织管理。那么我们可以认为这个`inode`就是`文件`。但`inode`无法记录文件名。这时我们就需要`链接`来帮忙了。


A link is just a pointer to an inode. A directory is an inode that holds links. Each filename in a directory is just a link to an inode. Opening a file in Unix also creates a link, but it's a different type of link (it's not a named link).

`链接`是一个指向`inode`的指针。`目录`是一个记录了若干链接的`inode`。目录下的每一个文件名都是一个指向`inode`的链接。在`Unix`中，打开一个文件就创建了一个`链接`，但这个`链接`不在这篇文章的讨论范围内，它是没有文件名的。


A hard link is just an extra directory entry pointing to that inode. When you ls -l, the number after the permissions is the named link count. Most regular files will have one link. Creating a new hard link to a file will make both filenames point to the same inode. Note:

硬链接是一个指向inode入口。观察`ls -l`的输出，权限字段后面的那个数字的含义是指向该inode的链接数。大多数普通文件只有一个链接。而创建一个硬链接会使两个文件名指向同一个inode。

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

Now, you can clearly see that there is no such thing as a hard link. A hard link is the same as a regular name. In the above example, test or test2, which is the original file and which is the hard link? By the end, you can't really tell (even by timestamps) because both names point to the same contents, the same inode:

现在，我们可以清楚的发现，根本没有`硬链接`这个东西。所谓的硬链接其实就是一个普通的文件名。在上面的例子中，我们根本没法分辨出test和test2哪个是原始文件那个是硬链接（连时间戳都一样）。因为，这两个文件名指向同样的inode，指向同样的数据。

```
% ls -li test*  
14445750 -rw-r--r--  2 danny  staff  0 Oct 13 17:58 test
14445750 -rw-r--r--  2 danny  staff  0 Oct 13 17:58 test2
14445892 -rw-r--r--  1 danny  staff  0 Oct 13 17:59 test3
```

The -i flag to ls shows you inode numbers in the beginning of the line. Note how test and test2 have the same inode number, but test3 has a different one.

参数`-i`用于显示文件的inode号。我们可以看到test和test2的inode号相同，而test3不用。


Now, if you were allowed to do this for directories, two different directories in different points in the filesystem could point to the same thing. In fact, a subdir could point back to its grandparent, creating a loop.

好了，如果我们为目录创建一个硬链接，那么位于文件系统中不同位置的两个目录就会指向同一内容。我们可以让子目录指向祖父目录，从而产生一个循环。


Why is this loop a concern? Because when you are traversing, there is no way to detect you are looping (without keeping track of inode numbers as you traverse). Imagine you are writing the du command, which needs to recurse through subdirs to find out about disk usage. How would du know when it hit a loop? It is error prone and a lot of bookkeeping that du would have to do, just to pull off this simple task.

为什么不能有循环呢？因为遍历文件树时，我们没法知道我们是不是在兜圈子（除非一边遍历一边记录inode号）。比如，`du`命令会递归地遍历所有子目录来计算磁盘使用量。而循环会让`du`陷入麻烦，它不得不做大量的记录工作来完成原本简单任务。


Symlinks are a whole different beast, in that they are a special type of "file" that many file filesystem APIs tend to automatically follow. Note, a symlink can point to a nonexistent destination, because they point by name, and not directly to an inode. That concept doesn't make sense with hard links, because the mere existence of a "hard link" means the file exists.

而`符号链接`（Symlinks）则是完全不同的一个物种了。它是一类特殊的文件类型，很多文件系统都支持。注意，符号链接可以指向一个不存在的文件，因为它指向文件名而非inode。硬链接则不能指向空文件，只要硬链接存在就意味着文件存在。


So why can du deal with symlinks easily and not hard links? We were able to see above that hard links are indistinguishable from normal directory entries. Symlinks, however, are special, detectable, and skippable!  du notices that the symlink is a symlink, and skips it completely!

那么为什么`du`能轻松处理目录的符号链接却不支持硬链接呢？经过上面的讨论我们知道，目录的硬链接和普通目录是同一个东西。而符号链接则是一种特殊的文件类型，`du`能够检测出来并在遍历文件树时跳过它！


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

## 背单词

|英文|含义|
|-|-|
|npm|nodejs packages manager|
|nrm|npm registry manager|


## 原文
<https://unix.stackexchange.com/questions/22394/why-are-hard-links-to-directories-not-allowed-in-unix-linux?from=timeline>


> 封面图： 天山 - 2015夏 - Sandii
