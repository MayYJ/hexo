---
title: zookeeper-study
date: 2019-03-02 13:04:32
tags: zookeeper
---

#### 安装和使用zookeeper

1. 下载

   官方下载非常慢，所以这里给个清华镜像源的下载地址

​       [zookeeper 下载地址](https://mirrors.tuna.tsinghua.edu.cn/apache/zookeeper/zookeeper-3.4.13/zookeeper-3.4.13.tar.gz)

2. 配置

   解压文件后 到 conf 目录下 将zoo_simple.cfg 修改为 zoo.cfg并修改里面的参数， 例如：

   ```properties
   dataDir=E:\\software\\zookeeper\\zookeeper-3.4.13\\data
   dataLogDir=E:\\software\\zookeeper\\zookeeper-3.4.13\\log
   ```

3. 启动

   进入bin目录下 如果是 Linux 就选择zkServer.sh启动；如果是 Windows 就选择 zkServer.cmd启动

4. 使用

   - 客户端基本命令

     ```text
     建立节点   create /zk  hello
     获得节点  get /zk 
     设置节点 set /zk hello2
     建立子节点  set /zk/subzk hello3
     输出节点目录 ls /zk
     删除节点  delete /zk
     ```

