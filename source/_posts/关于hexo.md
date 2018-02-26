---
title: 关于hexo
date: 2018-02-25 23:32:31
tags: hexo
categories: hexo
---

###linux

####环境的准备

- sudo apt-get install git 
- sudo apt-get install nodejs
- sudo apt-get install npm

---

####自己遇到的一些问题

#####npm安装过慢

- npm安装hexo过慢：用官网的`npm install hexo-cli -g`速度非常感人

用淘宝的npm分流，安装命令：

```
npm install -g cnpm --registry=https://registry.npm.taobao.org
```

安装完过后用法和之前一样，只不过把npm改为cnpm

##### 设置social链接时候可能出现的问题

- 在设置social链接的时候，记得一定要把social前面的#去掉，还有记得在设置什么参数的时候记得留一个空格

##### hexo d 时可能出现的问题

- 如果其他没有问题出现 ERROR Deployer not found: git问题，那么可以尝试 npm install hexo-deployer-git --save

---

####其它有用的链接

- http://chitanda.me/2015/06/11/tips-for-setup-hexo/