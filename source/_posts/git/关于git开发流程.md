---
title: 关于git开发规范
date: 2018-12-14 16:18:09
tags: git
category: git
---

开发的整个流程应该规范化，不然每一个人都用自己的一套就会让项目显得杂乱无章，让项目的进行不顺利，也让项目后期举步维艰，因为我们无法去定位代码问题；也让项目的迭代很困难，因为其它接手的同学看着乱糟糟的代码就头痛

在这里我也想说说什么叫项目的迭代，就是在现有项目的基础上添加一些新的功能

#### Git分支规范

##### 第一种

![](https://github.com/longtian2/cc3/blob/master/images/ali_svn_branch.png?raw=true)

有四类分支 master、dev_xxx、test_xxx、hotfix_xxx

**master**: 主分支，主要用来发布版本，每一次提交到master 分支都需要打tag

**dev_xxx**：开发分支，由迭代确定发布内容后创建开发分支，xxx指的是某个模块或者功能加上截止日期

**test_xxx**：迭代测试分支，由开发同学在自测通过后创建通过后创建该分支，然后由测试同学去测试上面的代码，有错误的话开发的同学在开发分支上进行修改bug，测试同学自行安排时间合并开发分支上的代码

**hotfix_xxx**：紧急修复分支，该分支主要是解决合并代码后，在sit环境、预付发布环境、线上发现bug时创建

##### 第二种

![](https://github.com/longtian2/cc3/blob/master/images/git-branch.png?raw=true)

**master**：主分支，主要用来发布版本，每一次提交到master 分支都需要打tag

**dev**：日常开发分支，要保证上线都是最新的和正确的代码

**feature**：功能分支，具体的某个功能的分支，至于dev分支交互且该分支只是本地的分支，当该分支功能测试通过后再合并到dev分支

**release**：该分支是用来做测试用的，当某个功能开发完成后并且合并到dev分支，就将该功能的commit cherry pick 到release 分支上做测试，如果有bug则直接在该分支上修改，待测试全部通过后再合并到dev分支和master分支

**hotfix**：进行线上修复bug的分支

#### Commit 规范

大致的格式如下：

```
type(scope): description..
```

type类型如下：

**feat**：提交新功能

**fix**：修补bug

**docs**：提交文档

**refactor**：重构代码

**test**：添加测试

**chore**：构建过程或辅助工具的变动

需要注意的是 只有feat 和 fix 出现在change log之中，其它不要出现其中；即提交用feat和fix type的代码 用git merge，而其它用git rebase