---
layout:     post
title:      "【翻译】为什么Unix/Linux不允许硬链接指向目录"
subtitle:   "Why are hard links to directories not allowed in UNIX/Linux?"
date:       2018-07-19 12:00:00
author:     "Danny Dulai & G-Man"
header-img: "img/banner/bg-004.jpg"
catalog: true
tags:
    - linux
---

## 问

I read in text books that Unix/Linux doesn't allow hard links to directories but does allow soft links. Is it because, when we have cycles and if we create hard links, and after some time we delete the original file, it will point to some garbage value?

教科书中说，Unix/Linux不允许硬链接指向目录，但软链接却可以。这是因为目录的硬链接会在文件系统中产生循环。而且如果我们给目录创建了硬链接之后再删除原目录，硬链接会指向垃圾数据。是这样的吗？

If cycles were the sole reason behind not allowing hard links, then why are soft links to directories allowed?

若只是因为文件系统循环就禁用目录硬链接的话，那为什么又允许指向目录的软链接呢？


## 答

This is just a bad idea, as there is no way to tell the difference between a hard link and an original name.

硬链接指向目录是行不通的。因为目录本身和它的硬链接。

Allowing hard links to directories would break the directed acyclic graph structure of the filesystem, possibly creating directory loops and dangling directory subtrees, which would make fsck and any other file tree walkers error prone.

First, to understand this, let's talk about inodes. The data in the filesystem is held in blocks on the disk, and those blocks are collected together by an inode. You can think of the inode as THE file.  Inodes lack filenames, though. That's where links come in.

A link is just a pointer to an inode. A directory is an inode that holds links. Each filename in a directory is just a link to an inode. Opening a file in Unix also creates a link, but it's a different type of link (it's not a named link).

A hard link is just an extra directory entry pointing to that inode. When you ls -l, the number after the permissions is the named link count. Most regular files will have one link. Creating a new hard link to a file will make both filenames point to the same inode. Note:

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
Now, you can clearly see that there is no such thing as a hard link. A hard link is the same as a regular name. In the above example, test or test2, which is the original file and which is the hard link? By the end, you can't really tell (even by timestamps) because both names point to the same contents, the same inode:

% ls -li test*  
14445750 -rw-r--r--  2 danny  staff  0 Oct 13 17:58 test
14445750 -rw-r--r--  2 danny  staff  0 Oct 13 17:58 test2
14445892 -rw-r--r--  1 danny  staff  0 Oct 13 17:59 test3
The -i flag to ls shows you inode numbers in the beginning of the line. Note how test and test2 have the same inode number, but test3 has a different one.

Now, if you were allowed to do this for directories, two different directories in different points in the filesystem could point to the same thing. In fact, a subdir could point back to its grandparent, creating a loop.

Why is this loop a concern? Because when you are traversing, there is no way to detect you are looping (without keeping track of inode numbers as you traverse). Imagine you are writing the du command, which needs to recurse through subdirs to find out about disk usage. How would du know when it hit a loop? It is error prone and a lot of bookkeeping that du would have to do, just to pull off this simple task.

Symlinks are a whole different beast, in that they are a special type of "file" that many file filesystem APIs tend to automatically follow. Note, a symlink can point to a nonexistent destination, because they point by name, and not directly to an inode. That concept doesn't make sense with hard links, because the mere existence of a "hard link" means the file exists.

So why can du deal with symlinks easily and not hard links? We were able to see above that hard links are indistinguishable from normal directory entries. Symlinks, however, are special, detectable, and skippable!  du notices that the symlink is a symlink, and skips it completely!

% ls -l 
total 4
drwxr-xr-x  3 danny  staff  102 Oct 13 18:14 test1/
lrwxr-xr-x  1 danny  staff    5 Oct 13 18:13 test2@ -> test1
% du -ah
242M    ./test1/bigfile
242M    ./test1
4.0K    ./test2
242M    .

## 背单词

|英文|含义|
|-|-|
|npm|nodejs packages manager|
|nrm|npm registry manager|


## 原文
<https://unix.stackexchange.com/questions/22394/why-are-hard-links-to-directories-not-allowed-in-unix-linux?from=timeline>


> 封面图： 天山 - 2015夏 - Sandii
