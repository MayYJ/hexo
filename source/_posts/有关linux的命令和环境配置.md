---
title: 有关linux的命令
date: 2018-02-25 23:40:27
tags: linux命令，环境
categories: linux
---

1. sudo: add-apt-repository：找不到命令

   solution：sudo apt-get install python-software-properties 

   ​                   sudo apt-get install software-properties-common

2. 当linux上没有什么快捷命令的时候我们可以自己在~/.bashrc中设置

   - vim ~/.bashrc设置别名
   - alias ll='ls -alF'
   - source ~/.bashrc

3. 配置java环境变量

   - 首先下载jdk

   - sudo tar zxvf  ./{jdk文件} -C  /usr/lib

   - cd /usr/lib

   - sudo mv {jdk文件} jdk8

   - sudo vi /etc/profile

   - 在最后加入

     ```
     export JAVA_HOME=/usr/lib/jdk7
     export JRE_HOME=${JAVA_HOME}/jre
     export CLASSPATH=.:${JAVA_HOME}/lib/tools.jar:${JRE_HOME}/lib/dt.jar
     export PATH=${JAVA_HOME}/bin:${JAVA_HOME}/jre/bin:$PATH
     ```

   - sudo source /ect/profile

   - 最后 通过 java -version 进行检查

4. 设置系统自启动程序

   1. 在使用了systemd作为启动器的系统（如较新版的deepin）中，如果没有rc.local就自己创建，然后在文件里面写下如下内容：

      ```
      #!/bin/bash
      # rc.local config file created by use

      把你需要执行的命令写在这里

      exit 0
      ```

      再赋予权限 sudo chmod +x /etc/rc.local，下次重启时systemd就会自动执行rc.local里面的命              令了

   2. 在~/.configure/autostart 目录下添加自启动命令，以代理工具 XX-Net 为例，假定其启动脚本位于~/Documents/XX-Net-3.3.1/start。

      ```
      [Desktop Entry]
      Type=Application
      Exec="~/Documents/XX-Net-3.3.1/start"
      Hidden=false
      NoDisplay=false
      X-GNOME-Autostart-enabled=true
      Name[en_IN]=XX-Net
      Name=XX-Net
      Comment[en_IN]=XX-Net
      Comment=XX-Net
      ```

      系统启动时会执行 `Exec`所指定的命令。

      ​