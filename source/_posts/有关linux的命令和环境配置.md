---
title: 有关linux的命令
date: 2018-02-25 23:40:27
tags: linux命令，环境
categories: linux
---

[比较好的linux基础命令总结](http://blog.csdn.net/wojiaopanpan/article/details/7286430)

##### 命令错误的情况

sudo: add-apt-repository：找不到命令

solution：sudo apt-get install python-software-properties 

​                   sudo apt-get install software-properties-common

----

##### 设置命令的快捷方式

- vim ~/.bashrc设置别名

- alias ll='ls -al'

  如果要执行多条命令中间用分好分隔就行

- source ~/.bashrc

---

#####配置java环境变量

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

---

#####设置系统自启动程序

###### rc.local自启动

- 在使用了systemd作为启动器的系统（如较新版的deepin）中，如果没有rc.local就自己创建，然后在文件里面写下如下内容：

```
#!/bin/bash
# rc.local config file created by use

把你需要执行的命令写在这里

exit 0
```

再赋予权限 sudo chmod +x /etc/rc.local，下次重启时systemd就会自动执行rc.local里面的命令了

###### autostart自启动

- 在~/.configure/autostart 目录下添加自启动命令，以代理工具 XX-Net 为例，假定其启动脚本位于~/Documents/XX-Net-3.3.1/start。

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

系统启动时会执行 `Exec`所指定的命令

---

##### 进程相关命令

###### 查询进程

- ps命令查找与进程相关的PID号：
- ps a 显示现行终端机下的所有程序，包括其他用户的程序。
- ps -A 显示所有程序。
- ps c 列出程序时，显示每个程序真正的指令名称，而不包含路径，参数或常驻服务的标示。
- ps -e 此参数的效果和指定"A"参数相同。
- ps e 列出程序时，显示每个程序所使用的环境变量。
-  ps f 用ASCII字符显示树状结构，表达程序间的相互关系。
-  ps -H 显示树状结构，表示程序间的相互关系。
- ps -N 显示所有的程序，除了执行ps指令终端机下的程序之外。
- ps s 采用程序信号的格式显示程序状况。
- ps S 列出程序时，包括已中断的子程序资料。
- ps -t<终端机编号> 指定终端机编号，并列出属于该终端机的程序的状况。
- ps u 以用户为主的格式来显示程序状况。
- ps x 显示所有程序，不以终端机来区分。

**最常用的方法是ps aux,然后再通过管道使用grep命令过滤查找特定的进程,然后再对特定的进程进行操作。**
**ps aux | grep program_filter_word,ps -ef |grep tomc**

---

###### 杀死进程

- 通过PID杀死进程

  ```
  kill PID
  ```

- 通过进程名称杀死进程

  ```
  killall NAME
  ```

  ​

##### 网络相关指令

###### netstat

- Netstat 命令用于显示各种网络相关信息，如网络连接，路由表，接口状态 (Interface Statistics)，masquerade 连接，多播成员 (Multicast Memberships) 等等。

- 相关参数

  - a (all)显示所有选项，默认不显示LISTEN相关
  - t (tcp)仅显示tcp相关选项
  - u (udp)仅显示udp相关选项
  - n 拒绝显示别名，能显示数字的全部转化成数字。
  - l 仅列出有在 Listen (监听) 的服務状态
  - p 显示建立相关链接的程序名
  - r 显示路由信息，路由表
  - e 显示扩展信息，例如uid等
  - s 按各个协议进行统计
  - c 每隔一个固定时间，执行该netstat命令。

  **提示：LISTEN和LISTENING的状态只有用-a或者-l才能看到**

###### 查看指定端口号的进程

```
# netstat -an | grep ':80'
```



