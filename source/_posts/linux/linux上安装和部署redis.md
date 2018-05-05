---
title: linux上安装和部署redis
date: 2018-03-09 19:37:40
tags: redis
categories: linux
---

1.基础知识
 redis是用C语言开发的一个开源的高性能键值对（key-value）数据库。它通过提供多种键值数据类型来适应不同场景下的存储需求，目前为止redis支持的键值数据类型如下
字符串、列表（lists）、集合（sets）、有序集合（sorts sets）、哈希表（hashs）
2.redis的应用场景
 缓存（数据查询、短连接、新闻内容、商品内容等等）。（最多使用）
 分布式集群架构中的session分离。
 聊天室的在线好友列表。
 任务队列。（秒杀、抢购、12306等等）
 应用排行榜。
 网站访问统计。
  数据过期处理（可以精确到毫秒）
3.安装redis
 下面介绍在Linux环境下，Redis的安装与部署，使用redis-3.0稳定版,因为redis从3.0开始增加了集群功能。在后面我也会分享redis集群。
 1.可以通过官网下载  地址：[http://download.redis.io/releases/redis-3.0.0.tar.gz](https://link.jianshu.com?t=http://download.redis.io/releases/redis-3.0.0.tar.gz)
 2.使用linux wget命令

```
wget http://download.redis.io/releases/redis-3.0.0.tar.gz
```

将redis-3.0.0.tar.gz拷贝到/usr/local下

```
cp redis-3.0.0.rar.gz /usr/local

```

解压源码

```
tar -zxvf redis-3.0.0.tar.gz 

```

进入解压后的目录进行编译

```
cd /usr/local/redis-3.0.0

```

安装到指定目录  如 /usr/local/redis

```
make PREFIX=/usr/local/redis install

```

redis.conf是redis的配置文件，redis.conf在redis源码目录。
拷贝配置文件到安装目录下
进入源码目录，里面有一份配置文件 redis.conf，然后将其拷贝到安装路径下

```
cd /usr/local/redis
mkdir conf
cp /usr/local/redis-3.0.0/redis.conf  /usr/local/redis/bin

```

进入安装目录bin下

```
cd /usr/local/redis/bin

```

此时我们看到的目录结构是这样的

![img](http://upload-images.jianshu.io/upload_images/5145552-5e2339176d6a2d63.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

redis-benchmark   redis性能测试工具
redis-check-aof     AOF文件修复工具
redis-check-rdb     RDB文件修复工具
redis-cli      redis命令行客户端
redis.conf   redis配置文件
redis-sentinal   redis集群管理工具
redis-server  redis服务进程

4.启动redis
 1.前端模式启动
直接运行bin/redis-server将以前端模式启动，前端模式启动的缺点是ssh命令窗口关闭则redis-server程序结束，不推荐使用此方法

```
./redis-server

```

如图

![img](https://upload-images.jianshu.io/upload_images/5145552-a3bbd113c8131572.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)



 2.后端模式启动
修改redis.conf配置文件， daemonize yes 以后端模式启动

```
vim /usr/local/redis/bin/redis.conf

```

![img](http://upload-images.jianshu.io/upload_images/5145552-6e99ad064ae73055.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

执行如下命令启动redis：

```
cd /usr/local/redis
./bin/redis-server ./redis.conf

```

连接redis

```
/usr/local/redis/bin/redis-cli 

```

5.关闭redis
强行终止redis进程可能会导致redis持久化数据丢失。正确停止Redis的方式应该是向Redis发送SHUTDOWN命令，命令为：

```
cd /usr/local/redis
./bin/redis-cli shutdown

```

强行终止redis

```
pkill redis-server

```

让redis开机自启

```
vim /etc/rc.local
//添加
/usr/local/redis/bin/redis-server /usr/local/redis/etc/redis-conf

```

至此redis已经全部安装完，后面我会分享redis.conf 详细配置以及说明。