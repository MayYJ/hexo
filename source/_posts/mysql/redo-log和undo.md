---
title: redo.log和undo.log
date: 2019-03-13 16:33:37
tags: redo.log,undo.log
---

#### undo.log

##### 概述

undo.log 实现了有两个功能：

1. 数据回滚
2. MVCC 的实现

需要注意的是undo.log是逻辑日志，所以对于数据的回滚它不是通过物理的恢复到执行语句或事务之前的样子，而是逻辑的把前面的修改都取消了，即使用了相反的SQL语句

下面我将重点讲Innodb MVCC的实现，在讲解其实现之前先要有下面的知识点：

##### 数据存储

1. 在《高性能Mysql》中提到了其实现是通过添加了两个隐藏列，一个创建版本号一个删除版本号，而这两个版本号其实是通过undo.log 的指针得到的

2. 所以这里又得提到InnodB 聚簇索引存储数据的格式，它会在每行数据后面添加两个字段：一个时6字节的事务ID(`DB_TRX_ID`)字段，用于标记最近修改该数据的事务ID，每处理一个事务，该值加1；7字节的回滚指针(`DB_ROLL_PTR`)字段，指向当前记录项的rollback segment的undo.log，找之前版本的数据就是通过这个指针

   ![](https://pic3.zhimg.com/80/v2-e1844f5816a332018183559d1573d80e_hd.jpg)

##### undo.log 的存储管理

1. undo.log的存储管理：Innodb 对 undo.log的管理采用段的方式，Innodb存储引擎有rollback segment，每个回滚段中记录了1024个undo log segment，而在每个undo log segment段中进行undo页的申请；每个事务Commit了不会立即将undo.log删除了，而是将其放入一个链表中，等待purge线程来判断什么时候删除；

   undo.log 里面会记录每个事务修改的记录原始的值；所以对于增加、删除、以及修改就会有不同的行为，undo.log 分为Insert undo.log 和  update undo log； 对于添加操作添加到insert undo log，在提交完成后会直接删除，因为事务的隔离性其它事务在该事务提交前是不能看到添加的数据所以也就对于MVCC没有意义，它的作用就是用于回滚；修改和删除操作都是记录到update undo log，记录的新数据的备份，以便其它的记录能够看到；删除操作只是作一个标记，最终删除操作是要等purge线程判断可以删除该undo页的时候再真实的删除

##### 事务链表

MySQL中的事务在开始到提交这段过程中，都会被保存到一个叫trx_sys的事务链表中，这是一个基本的链表结构：

![](https://pic2.zhimg.com/80/v2-d750223af8b18e8c9eb65cc4a9856ec1_hd.jpg)

事务链表中保存的都是还未提交的事务，事务一旦被提交，则会被从事务链表中摘除。

##### ReadView

有了前面隐藏列和事务链表的基础，接下去就可以构造MySQL实现MVCC的关键——ReadView。

ReadView说白了就是一个数据结构，在SQL开始的时候被创建。这个数据结构中包含了3个主要的成员：ReadView{low_trx_id, up_trx_id, trx_ids}，在并发情况下，一个事务在启动时，trx_sys链表中存在部分还未提交的事务，那么哪些改变对当前事务是可见的，哪些又是不可见的，这个需要通过ReadView来进行判定，首先来看下ReadView中的3个成员各自代表的意思：

1. low_trx_id表示该SQL启动时，当前事务链表中最大的事务id编号，也就是最近创建的除自身以外最大事务编号；
2. up_trx_id表示该SQL启动时，当前事务链表中最小的事务id编号，也就是当前系统中创建最早但还未提交的事务；
3. trx_ids表示所有事务链表中事务的id集合。

上述3个成员组成了ReadView中的主要部分，简单图示如下：

![](https://pic1.zhimg.com/80/v2-7b3dc9ba4be387f086fc63f114031574_hd.jpg)

根据上图所示，所有数据行上DATA_TRX_ID小于up_trx_id的记录，说明修改该行的事务在当前事务开启之前都已经提交完成，所以对当前事务来说，都是可见的。而对于DATA_TRX_ID大于low_trx_id的记录，说明修改该行记录的事务在当前事务之后，所以对于当前事务来说是不可见的。

**注意，ReadView是与SQL绑定的，而并不是事务，所以即使在同一个事务中，每次SQL启动时构造的ReadView的up_trx_id和low_trx_id也都是不一样的，至于DATA_TRX_ID大于low_trx_id本身出现也只有当多个SQL并发的时候，在一个SQL构造完ReadView之后，另外一个SQL修改了数据后又进行了提交，对于这种情况，数据其实是不可见的。**

最后，至于位于（up_trx_id, low_trx_id）中间的事务是否可见，这个需要根据不同的事务隔离级别来确定。对于RC的事务隔离级别来说，对于事务执行过程中，已经提交的事务的数据，对当前事务是可见的，也就是说上述图中，当前事务运行过程中，trx1~4中任意一个事务提交，对当前事务来说都是可见的；而对于RR隔离级别来说，事务启动时，已经开始的事务链表中的事务的所有修改都是不可见的，所以在RR级别下，low_trx_id基本保持与up_trx_id相同的值即可。

##### MVCC实现图示过程

![](https://user-gold-cdn.xitu.io/2018/1/2/160b63c130c3c306?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

#### redo.log

redo.log 称为重做日志，用来保证事务的原子性和持久性

##### 与binlog的区别

redo.log 是Innodb 存储引擎层产生的，是物理格式日志记录的是对相应页的修改；而binlog是在Mysql数据库的上层产生的，其是一种逻辑日志，记录的是其对应的sql语句

binlog 只在事务commit的时候记录，redo.log由于会记录undo.log的日志所以它会在事务进行过程中记录事务的多个条目

redo.log 在加上checkPoint机制，能够实现崩溃恢复；binlog 用来进行POINT-IN-TIME的恢复及主从复制环境的建立

下面我用一张图来表示redo.log 记录传输过程

![](https://s2.ax1x.com/2019/03/13/AkgKq1.png)

其实Innodb里面对所有数据的操作都满足这样的三个步骤；只不过这里不一样的是用LSN对这些步骤进行了连接，三个步骤的LSN 分别对应了Innodb的三个参数 Log sequence number 表示当前的LSN，Log flushed up to 表示刷新到重做日志文件的LSN，Last checkPoint at 表示刷新到磁盘的LSN；通过这个LSN也给崩溃恢复提供了沃土

#### 参考

MVCC 的实现

[https://zhuanlan.zhihu.com/p/40208895](https://zhuanlan.zhihu.com/p/40208895)

[https://juejin.im/entry/5a4b52eef265da431120954b](https://juejin.im/entry/5a4b52eef265da431120954b)

《Mysql技术内部》

