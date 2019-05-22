---
title: 面向实习的java后台学习知识点（待完善）
date: 2018-03-12 14:59:50
tags: 学习路线
---

#### java基础

- 集合框架 hashmap、linkedlist、arraylist

- 线程安全 vector、concurrenthash、hashtable

- 线程安全 

  - 关键字： syncronized、 violate 、reenterlock

- jvm锁优化 自旋锁.....

- String

  - 常量池：运行时常量池、静态常量池
  - final ----> 并发
  - StringBUffer、StringBuilder

- 线程

  - 很杂（多线程面经）
  - 线程池
    - 有哪些
    - 阻塞队列
    - 拒绝策略

- io

  - BIO

  - NIO

    **nio在RPC框架中的作用**

- object

  - toString
  - hushcode
  - equals
  - 深克隆和浅克隆
  - wait 和sleep()
  - notify 和notifyall

#### JVM

- 内存区域
- 内存分配
- 内存回收 (GC 和 GC算法)
- class文件结构
- 内存模型(线程)
  - 线程实现
  - 线程优化
  - 类加载器

#### Spring

- **IOC**(很重要的点，需要了解底层)
- **AOP**（很重要的点，需要了解底层）
- 事务

#### mybaties

- 怎么实现事务

#### spring mvc

- SpringBeanFactory

- spirngMVC处理流程

- springmvc容器初始化

  mysql


- 事务（四个特性、隔离级别、读问题）
- 并发处理(悲观锁、乐观锁)
- 索引(betry，hash、全文、空间)
- 优化(sql优化、分库分表(mycat) )

#### 计算机网络

-  http：分层、https 数字含义
-  tcp：三次握手、四次挥手
- udp
- cookie、session

#### 操作系统

- 调度算法
- cpu
- 大文件的排序

#### 数据结构

- 数组(内存形式)--->

链表(栈(递归调用)、队列)--->树(二叉树、完全二叉树、排序二叉树、平衡二叉树(B树)、遍历过程（中序、）)--->图(因为现在还没有学习数据结构，学习过后再来总结这里的大的知识点)

#### 算法

- 查找
- 排序


- 动态规划(贪婪算法)

#### 项目

- RPC(三宗罪、nio)
- 负载均衡(无状态服务)
- 注册中心(nio、路由算法、缓存(雪崩、扩展、内存/jvm 缓存))
- 数据库缓存集群（读写分离、分库分表、高可用高扩展）

#### Redis

了解其基本实现