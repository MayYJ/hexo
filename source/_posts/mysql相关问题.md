---
title: mysql相关问题
date: 2018-02-27 15:16:17
tags: mysql
categories: mysql
---

#### 增加远程登录权限

- 本地登录mysql后执行一下命令

  ```
  grant all privileges on *.* to root@'%' identified by 'root';
  flush privileges;
  ```

  ​