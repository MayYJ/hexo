---
title: git命令
date: 2018-02-27 00:03:19
tags: git命令
categories: git
---

#### git reset

**--soft**: 将Head引用指向给定提交，索引和工作目录的内容都不会表，这个功能就类似于git checkout

**--mixed**: 将HEAD 引用指向给定提交，索引内容跟着改变，但是工作目录内容不会改变

**--hard**：将HEAD引用指向给定提交，索引内容和工作目录内容都跟着改变(所以这个需要慎用，因为一旦使用这个命令两个commit之间所有文件都会被删除)

#### git rebase 和 git merge

![](https://upload-images.jianshu.io/upload_images/305877-5dece524b7130343.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)

![](https://upload-images.jianshu.io/upload_images/305877-c4ddfcf679821e2f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)

![](https://upload-images.jianshu.io/upload_images/305877-467ba180733adca1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)

**解释**

从上面的三张图就可以大致看出rebase和merge的区别：merge是在自己的分支新建一个commit而这个commit里面包含了master的所有新的提交；rebase是直接将master 的新的commit链接在当前分支的尾部

**问题**

![](https://upload-images.jianshu.io/upload_images/305877-8845daa6b5fd004e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)

git 黄金法则：绝对不要再公共分支上使用git rebase

**两个命令解决冲突的方法**

merge：发现冲突，提交冲突的解决，再合并

rebase：修改冲突，git add，git rebase --continue

#### git stash 

该命令想要解决的是当我们一个功能写到一半的时候，但是现在出现了另外一个更加急迫的功能需要先去完成，但是因为当前功能并没有完成，我们不想直接提交，这个时候就需要用到这个命令