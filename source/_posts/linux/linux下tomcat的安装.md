---
title: linux下tomcat的安装
date: 2018-02-25 23:37:42
tags: tomcat的安装
categories: linux 
---

1. 下载[tomcat](https://tomcat.apache.org/download-90.cgi)压缩文件

   tar -zxvf {tomcat文件}

   移动到/usr/local 文件下 并把文件名称改为类似tomcat9

2. 配置环境变量

   vi /etc/profile

   在文件最后加入

   ```
   export CATALINA_HOME=/usr/local/tomcat9
   ```

   source /etc/profile

3. 配置tomcat的catAlina.sh文件

   cd $CATALINA_HOME/bin

   vi catalina.sh

   找到OS specific support 然后在下面添加

   ```
   CATALINA_HOME=/usr/local/tomcat9
   JAVA_HOME=jdk文件所在位置
   ```

4. 安装tomcat服务

   cp catalina.sh /etc/init.d/tomcat

   chkconfig --add tomcat

   chkconfig tomcat on

   chkconfig --liist  查看tomcat是否添加服务成功

5. 权限问题

   - cd $CATALINA_HOME/bin
   - sudo chmod 777 *.sh
   - cd /etc/init.d
   - sudo chmod 777 tomcat