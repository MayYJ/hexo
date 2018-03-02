---
title: linux装windwos后修复MBR
date: 2018-03-03 01:51:25
tags: MBR
categories: linux
---

##### 原因

在linux系统下安装windows系统，windows系统比较霸道，会覆盖掉所有的MBR文件，所有下次启动的时候不会使你选择启动的系统而是直接进入windows系统。这个时候就需要修复引导文件。

##### 修复引导文件

1. 用U盘做一个linux的启动盘

2. 用启动盘进入试用

3. 打开命令行终端

4. sudo add-apt-repository ppa:yannubuntu/boot-repair && sudo apt-get update

   sudo apt-get install -y boot-repair && boot-repair

5. 出现的界面里面选择Recommended repair

   使用完成就完成了

##### 注意

- 如果有些人不小心点击了Create a BootInfo summary的话，那你的开机启动界面将会出来一大堆你以前没见过的东西。

   那样的话，你可以输入名令：cd /boot/grub

  接着输入sudo gedit grub.cfg,打开grub.cfg文件后，通过搜索找到windows，然后把下面这些删去就和原来一样了。