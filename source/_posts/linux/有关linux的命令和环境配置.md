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

---

##### 安装deb包

- 下面是一个例子：

```
sudo dpkg -i Appifier_9.6.2_amd64.deb
```

##### 实时跟踪tomcat的输出

1. 进入tomcat的logs目录

   例如：

   ```
   cd /usr/local/tomcat8/logs
   ```

2. ```
   tail -f catalina.out
   ```


##### linux下解压所有类型压缩类型文件的命令

.tar 
解包：tar xvf FileName.tar
打包：tar cvf FileName.tar DirName
（注：tar是打包，不是压缩！）
———————————————
.gz
解压1：gunzip FileName.gz
解压2：gzip -d FileName.gz
压缩：gzip FileName

.tar.gz 和 .tgz
解压：tar zxvf FileName.tar.gz
压缩：tar zcvf FileName.tar.gz DirName
———————————————
.bz2
解压1：bzip2 -d FileName.bz2
解压2：bunzip2 FileName.bz2
压缩： bzip2 -z FileName

.tar.bz2
解压：tar jxvf FileName.tar.bz2
压缩：tar jcvf FileName.tar.bz2 DirName
———————————————
.bz
解压1：bzip2 -d FileName.bz
解压2：bunzip2 FileName.bz
压缩：未知

.tar.bz
解压：tar jxvf FileName.tar.bz
压缩：未知
———————————————
.Z
解压：uncompress FileName.Z
压缩：compress FileName
.tar.Z

解压：tar Zxvf FileName.tar.Z
压缩：tar Zcvf FileName.tar.Z DirName
———————————————
.zip
解压：unzip FileName.zip
压缩：zip FileName.zip DirName
———————————————
.rar
解压：rar x FileName.rar
压缩：rar a FileName.rar DirName
———————————————
.lha
解压：lha -e FileName.lha
压缩：lha -a FileName.lha FileName
———————————————
.rpm
解包：rpm2cpio FileName.rpm | cpio -div
———————————————
.deb
解包：ar p FileName.deb data.tar.gz | tar zxf -
———————————————

.tar .tgz .tar.gz .tar.Z .tar.bz .tar.bz2 .zip .cpio .rpm .deb .slp .arj .rar .ace .lha .lzh .lzx .lzs .arc .sda .sfx .lnx .zoo .cab .kar .cpt .pit .sit .sea
解压：sEx x FileName.*
压缩：sEx a FileName.* FileName

sEx只是调用相关程序，本身并无压缩、解压功能，请注意！

gzip 命令 
减少文件大小有两个明显的好处，一是可以减少存储空间，二是通过网络传输文件时，可以减少传输的时间。gzip 是在 Linux 系统中经常使用的一个对文件进行压缩和解压缩的命令，既方便又好用。

语法：gzip [选项] 压缩（解压缩）的文件名该命令的各选项含义如下：

-c 将输出写到标准输出上，并保留原有文件。-d 将压缩文件解压。-l 对每个压缩文件，显示下列字段：     压缩文件的大小；未压缩文件的大小；压缩比；未压缩文件的名字-r 递归式地查找指定目录并压缩其中的所有文件或者是解压缩。-t 测试，检查压缩文件是否完整。-v 对每一个压缩和解压的文件，显示文件名和压缩比。-num 用指定的数字 num 调整压缩的速度，-1 或 --fast 表示最快压缩方法（低压缩比），-9 或--best表示最慢压缩方法（高压缩比）。系统缺省值为 6。指令实例：

gzip *% 把当前目录下的每个文件压缩成 .gz 文件。gzip -dv *% 把当前目录下每个压缩的文件解压，并列出详细的信息。gzip -l *% 详细显示例1中每个压缩的文件的信息，并不解压。gzip usr.tar% 压缩 tar 备份文件 usr.tar，此时压缩文件的扩展名为.tar.gz。

##### vim使用技巧

###### 使用关键字查询

使用‘/’加你要查询的内容

##### Center OS7

1. center os7安装后默认是安装了并启用了firewalld防火墙

2. 所以以下是一些命令操作该防火墙基本命令，有其它需求自行在网上搜索

   ```
   启动防火墙
   systemctl start firewalld 
   禁用防火墙
   systemctl stop firewalld
   设置开机启动
   systemctl enable firewalld
   停止并禁用开机启动
   sytemctl disable firewalld
   重启防火墙
   firewall-cmd --reload	
   查看状态
   systemctl status firewalld或者 firewall-cmd --state
   ```


##### ubuntu下快速构建java 环境

1. 安装jre

   ```
   sudo apt-get install default-jre
   ```

2. 安装jdk

   ```
   sudo apt-get install default-jdk
   ```

3. 下面是全面的安装

   ```
   sudo add-apt-repository ppa:webupd8team/java 

   sudo apt-get update 

   sudo apt-get install oracle-java8-installer 

   sudo apt-get install oracle-java8-set-default
   ```

   ​