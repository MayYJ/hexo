---
title: git命令
date: 2018-02-27 00:03:19
tags: git命令
categories: git
---

##### 删除远程仓库中的文件或者文件夹

```
git rm -r -n --cached  */src/\* 
```

这个命令是删除远程仓库的src目录下的所有文件， -n 加上这个这个参数是执行命令时预览要删除的文件，但是这个命令不会执行删除操作。

所以最终执行的是：

```
git rm -r --cached  */src/\* 
```



然后再 git commit， git push 就完成了远程仓库文件的删除。