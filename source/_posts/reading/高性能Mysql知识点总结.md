### MYSQL 基本架构

![](https://s2.ax1x.com/2019/02/10/kUdZVJ.jpg)

##### Mysql的三层逻辑结构

1. 第一层连接器用于连接处理，授权认证，安全管理
2. 第二层查询缓存、解析器、优化器是mysql中的服务器层，大多数Mysql的核心服务都在这一层，包括查询解析、分析、优化、缓存以及所有的内置函数，还有存储过程、触发器、视图等
3. 第三层是存储引擎层，存储引擎负责Mysql中的数据的存储和提取

##### 并发控制

Mysql在两个层面存在并发控制：服务器层和存储引擎层

**锁类型**：排它锁、共享锁

**锁粒度**：

1. 表级锁：是Mysql最基本的锁策略，并且是开销最小的策略，是Mysql服务器层的锁集机制
2. 行级锁：可以最大层度的支持并发处理（同时也带来了最大的锁开销）

##### 事务

定义：一组原子性的SQL查询，或者说一个独立的工作单元

1. **事务特性**

2. **隔离级别**

3. **死锁**：为了解决死锁，数据库系统实现了各种死锁检测和死锁超时机制；InnnoDB存储引擎目前处理死锁的方法是，将持有最少行级排他锁的事务进行回滚

4. **事务日志**：事务日志可以帮助提高事务的效率；使用事务日志，存储引擎在修改表的数据时只需要修改其内存拷贝，再把该行为记录到持久在硬盘上的事务日志中(事务日志采用追加方式是顺序IO，所以会比较快)；事务日志持久之后，内存中被修改的数据在后台可以慢慢刷回到磁盘；这种方式通常叫做预写式日志

   Innodb主要通过事务日志实现ACID特性，事务日志包括：重做日志redo和回滚日志undo

   Redo记录的是已经全部完成的事务，就是执行了Commit的事务

   undo记录的是已部分完成并且已经写入硬盘的未完成的事务

   能够实现系统崩溃后，重启后能够自动恢复数据

5. **Mysql中的事务**：mysql默认采用自动提交模式即每个查询都会被当做一个事务进行提交

   两阶段锁定协议：在事务执行过程中，随时都可以锁定，在Commit或者Rollback时才会被释放，并且所有锁在同一时刻被释放

   隐式锁：Innodb根据隔离级别自动加锁

   显示锁：通过特定语句显示的加锁

##### 多版本并发控制

MVCC是行级锁的变种，很多情况下会避免加锁操作

InnoDB的MVCC是通过在每行记录后面保存两个隐藏的列来实现的，这两个列，一个保存了创建时间，一个保存的删除时间；当然这里的时间并不是真正的时间而是系统事务的版本号，当新建一个事务，事务版本号就会递增加1

##### MyISAM和InnoDB的差异

1. Innodb支持事务 MyISAM不支持
2. Innodb 支持外键 MyISAM不支持
3. Innodb 使用聚簇索引 MyISAM 使用非聚簇索引
4. Innodb 具有崩溃恢复机制 MyISAM没有
5. Innodb支持行级锁 MyISAM不支持

##### Mysql 中索引类型

1. 哈希索引

   哈希索引基于哈希表，对于每一行存储引擎都使用所有索引列计算一个哈希码

2. 全文索引

   全文索引是目前搜索引擎使用的一种关键技术，它能够利用分词技术等多种算法智能分析出文本中关键字词出现的频率及重要性，然后按照一定的算法规则智能筛选出我们想要的搜索结果

### 剖析Mysql查询的方法

##### 剖析单条查询的方法

1. show profiles：

   set profiling = 1；

   show profiles；

   show profile for query ?;

   第一条代码开启分析查询语句时间功能

   第二条代码查看所有查询语句总耗时

   第三条代码查看每条查询语句执行的每个阶段耗时

### Schema与数据类型优化

1. 选择优化得数据类型原则：

   - 更小的通常更好
   - 简单就好
   - 尽量避免NULL，因为可为NULL的列使得索引，索引统计和值比较都更为复杂

2. 尽量不要使用Decimal，因为其会比DOUBLE或者FLOAT占用更大的空间且计算开销，除非你真的需要精确的小数计算比如存储财务数据

3. 关于VARCHAR和CHAR

   **VARCHAR**

   它比定长更节省空间，因为它仅使用必要的空间；所以在以下情况下最好使用该类型：

   - 字符串列的长度比平均长度大很多
   - 列的更新很少，所以碎片不是问题(因为VARCHAR的更新可能会导致原来的页剩余的空间不够而需要采取其它的技术手段)
   - 使用了UTF-8这样复杂的字符集，每个字符使用不同的字节数进行存储

   **CHAR**

   Mysql总是会根据定义的字符串长度分配足够的空间，它会删除末尾的空格

   使用情况：

   - 很短的字符串且定长
   - 经常变更的数据

4. 关于DATETIME和TIMESTAMP

   TIMESTAMP会比DATETIME是更好的选择，因为前者会使用比后者一半对的存储空间但是它前者能够存储的时间范围比后者小

5. 关于Blob和Text

   Blob和Test的差别仅有分别采用二进制和字符串方式存储；它们不能对整个数据进行索引，而是截取前面的一些字段进行索引

### Mysql 调优

#### 创建高性能的索引

##### B-Tree索引

- 可以使用B-tree索引的查询类型
  1. 全值匹配：指的是和索引中的所有列进行匹配
  2. 匹配最左前缀
  3. 匹配列前缀
  4. 匹配范围值
  5. 精确匹配某一列并范围匹配另外一列
  6. 只访问索引的查询

- B-TREE索引的限制
  1. 如果不是按照索引的最左列开始查找，则无法使用索引
  2. 不能跳过索引中的列
  3. 如果查询中有某个列的范围查询，则右边所有列都无法使用索引优化查找

##### 索引的优点

1. 索引大大减少了服务器需要扫描的数据量

   快速定位，所以不需要导入全部数据进行顺序查找

2. 索引可以帮助服务器避免排序和临时表

   顺序存储，便于ORDER BY、GROUP BY ；当需要排序时，不是顺序存储的如果数据表小的话就把数据导入内存不然就用磁盘然后建立一个临时表进行排序这是一个非常耗时间的过程

3. 索引可以将随机IO变为顺序IO

   因为数据在索引中顺序存放的

##### 三星索引

三星索引其实就是对应的实现上面索引的三个优点；

通过一个例子来说明：

```sql
select A,B,C,D from user where A="xx" and B = "xx" order by C;
```

A 的选择性为0.01%

B 的选择性为0.1%

最佳的索引是（A,B,C,D），这是一个三星索引

- 第一颗星：过滤尽可能多的行，这意味着把选择性高的索引放在前面A,B；减少索引片的大小，以减少需要扫描的数据行
- 第二颗星：经过了A，B的筛选之后，筛选出来的行本身就是有序的；避免排序，减少磁盘IO和内存使用
- 第三颗星：通过宽索引实现覆盖索引；避免每一个索引对应的数据行都需要进行一次随机IO从聚集索引中读取数据

#### 高性能的索引策略

##### 独立的列

如果查询中的列不是独立的，则Mysql就不会使用索引；"独立的列"是指索引列不能是表达式的一部分，也不能是函数的一部分

ex：select actor_id from actor where actor_id + 1 = 5

##### 前缀索引和索引选择性

前缀索引：索引开始的部分字符，这样可以大大节约索引空间，从而提高索引效率。但是这样也会降低索引的选择性

索引的选择性：不重复的索引值和数据表的记录总数(#T)的比值

怎样确定索引前多少个字符?

```sql
select count(distinct left(city,3))/count(*) as sel3,
count(distinct left(city,4))/count(*) as sel4,
count(distinct left(city,5))/count(*) as sel5,
count(distinct left(city,6))/count(*) as sel6,
count(distinct left(city,7))/count(*) as sel7
```

选择选择性增加幅度很小的情况，但是也要注意数据分布不均匀的情况，即最常出现的前缀次数比最常出现的城市次数大很多的情况

前缀索引的缺点：无法使用前缀索引做ORDER BY 和 GROUP BY

##### 多列索引

举个例子：select film_id,actor_id from actor where actor_id='1' and film_id='1'；但是只有两个独立的分别是film_id的和actor_id的索引，那么mysql在查询时会使用"索引联合"策略，但是这个会有性能的问题，需要耗费大量CPU和内存资源在算法的缓存、排序和合并操作上；索引在EXPLAIN中有索引合并应该好好检查一下查询和表结构

##### 选择合适的索引列顺序

1. 将选择性最高的列索引放到最前列
2. 根据那些运行频率最高的插叙来调整索引列的顺序

##### 聚簇索引

聚簇：表示数据行和相邻的键值紧凑地存储在一起

聚簇索引包含了一张表的全部数据，一张表只能一个聚簇索引

优点：

1. 可以把相关数据保存在一起(通过索引我们只需读取少量有我们想要数据的数据页，而如果是MYISAM，就是通过索引得到想要数据的物理地址，读取一行就是一次磁盘IO)
2. 数据访问很快
3. 使用覆盖索引扫描的查询可以直接使用页结点中的主键值

缺点：

1. 最大限度提高了IO密集型应用的性能，但如果数据在内存中就没有用了
2. 插入速度严重依赖于插入顺序
3. 更新聚簇索引代价很高
4. 插入行可能有页分裂的问题
5. 聚簇索引可能导致全表扫描变慢
6. 二级索引可能比想象的要更大，因为二级索引的叶子结点包含了引用行的主键
7. 二级索引访问需要二次索引查找，而不是一次(如果你select 的字段比二级索引多，然后过滤条件能使用此二级索引，第一次先用二级索引进行查找找到对应的主键，然后通过主键在聚簇索引中找到想要的数据)

##### Innodb和MyISAM数据分布差异

- MyISAM数据分布

1. 主键索引和普通索引没有区别；

2. 按照数据插入的顺序插入的顺序存储在磁盘上，在索引数据行的旁边显示行号，从0开始递增，索引形式如图：

   ![](https://ws1.sinaimg.cn/large/a67bf22fgy1fu17m1cy9yj20ru09uq6f.jpg)

   即叶子结

- Innodb数据分布

  就用下面的图来标识吧，很清晰了

  ![](https://ws1.sinaimg.cn/large/a67bf22fgy1fu17roy983j20kv0fy7b1.jpg)

##### 覆盖索引

- 定义：如果一个索引包含所有需要查询的字段值，我们就称之为"覆盖索引"

##### 全表扫描和全索引扫描

全表扫描：是指通过物理表获取数据，顺序读磁盘上的数据

全索引扫描：查询时，遍历索引树来获取数据行

##### 使用索引扫描来做排序

Mysql有两种方式生成有序结果：通过排序操作或者按索引顺序扫描；所以如果直接通过排序操作，就可能用到中间表用来存储获取的数据然后通过算法排序，这样效率比较低

使用索引排序的要求：

1. 索引的列顺序和ORDER BY字句的顺序完全一致，并且所有列的排序方向都一样时。如果查询需要关联多张表，则只有当ORDER BY字句引用的字段全部为第一个表时，才能使用索引做排序
2. 索引的第一列被只能为一个常数，也能使用索引排序

##### 冗余和重复索引

Mysql 需要单独维护重复的索引，并且优化器在优化查询的时候也需要逐个的进行考虑，这会影响性能

##### 未使用的索引

##### 索引和锁

Innodb只有在访问行的时候才会对其加锁，而索引能够减少Innodb访问的行数，从而减少锁的数量

#### 查询性能优化

##### 为什么查询速度慢

在每一个消耗大量时间的查询案例中，我们都能看到一些不必要的额外操作，某些操作被额外的重复了很多次、某些操作执行得太慢等。优化查询的目的就是减少和消除这些操作所花费的时间

##### 慢查询基础：优化数据访问

查询性能低下的最基本原因是访问的数据太多

- 确认应用程序是否在检索大量超过需要的数据即列数是否是你需要的
- 确认Mysql服务器层是否在分析大量超过需要的行

1. 是否向数据库请求了不需要的数据

   - 查询不需要的记录
   - 多表关联时返回全部列
   - 总是取出全部列
   - 重复查询相同的数据

2. Mysql是否在扫描额外的记录

   衡量查询开销的三个指标如下：

   1. 响应时间：等待时间和响应时间的总和

   2. 扫描的行数

   3. 返回的行数

      - 扫描的行数和返回的行数：理想情况下扫描的行数和返回的行数相同；如果我们使用了索引，那么就会先从索引中得到我们需要的数据的主键，然后使用这个主键去聚簇索引中得到需要的数据，这样也就减少了扫描的行数；如果是全表扫描，那么就会把全部数据加载进内存，然后一个一个跟条件比较得到结果

      - 扫描的行数和访问类型(type)

        访问类型：扫描表、扫描索引、范围访问和单值访问

        索引让Mysql以最高效、扫描行数最少的方式找到需要的记录

      - 如果存在扫描的行数远大于返回的行数，一般的解决方式如下；

        1. 使用索引覆盖扫描，把所有需要用到的列都放到索引中
        2. 改变库表结构：例如使用单独的汇总表
        3. 重写这个复杂的查询，让Mysql优化器以更优化的方式去执行

##### 重构查询方式

1. 一个复杂查询还是多个简单查询
2. 切分查询
3. 分解关联查询

##### 查询执行的基础

Mysql发送一个请求的时候，Mysql到底做了些什么：

1. 客户端发送 一条查询给服务器
2. 服务器先检查查询缓存，如果命中了缓存，则立刻返回存储在缓存中的结果。否则进入下一阶段
3. 服务器进行SQL解析、预处理，再由优化器生成对应的执行计划
4. Mysql根据优化器生产的执行计划、调用存储引擎的API来执行查询
5. 将结果返回给客户端

- Mysql客户端/服务器通信协议

- 查询状态

  1. Sleep

     线程正在等待客户端

  2. Query

     线程正在执行查询或者正在将结果发送给客户端

  3. Locked

     在Mysql服务器层，该线程正在等待表锁

  4. Analyzing and statistics

     线程正在收集存储引擎的统计信息，并生成查询的执行计划

  5. Copying to tem table

     线程正在执行查询，并且将其结果集收集都复制到一个临时表中

  6. Storing result

     线程正在对结果集进行排序

  7. Sending data

     线程可能在多个状态之间传递数据，或者在生成结果集，或者在向客户端返回数据

##### 优化特定类型的查询

1. 优化COUNT()查询

   - COUNT()的作用：是它可以统计某个列值的数量，也可以统计行数，在统计值时要求列值是非空的
   - 关于MyISAM的神话： MyISAM的COUNT()函数总是非常快，这是个误解，只有在没有任何WHERE条件的COUNT(*)才非常快
   - 简单的优化

2. 优化LIMIT分页

   为什么单纯的LIMIT性能不好，因为mysql会扫描全部数据，丢弃前面无用数据，然后只返回LIMIT的数据量

   - 把LIMIT更改为使用 BETWEEN ... AND

   - 记录上一次查询返回的id，下一次从该id开始查询

     ex：select * from rental where rental_id < 16030 order by rental_id limit 201

### Innodb存储引擎

##### Innodb 锁机制

###### Mysql 查看锁的操作

1. 通过命令show engine	innodb status命令查看当前锁请求信息
2. 在infomation_schema 架构下添加了表Innodb_trx、innodb_locks、innodb_lock_waits；通过这三张表，用户可以更加简单地监控当前事务并分析可能存在的锁问题

###### 共享锁和排他锁

- 一个共享锁允许事务持有这个锁来读取数据
- 一个排它锁允许事务持有这个锁来修改数据

意向锁

这个锁允许事务行级锁和表级锁同时存在

- 意向共享锁：事务想要获得一张表中某几行的共享锁
- 意向排它锁：事务想要获得一张表中某几行的排它锁

举例来说：在对记录r加X锁之前，已经有事务对表1进行了S表锁，那么表1上已经存在S锁，之后事务需要对记录r在表上加上IX，由于不兼容，所以该事务需要等待表锁操作的完成

###### 记录锁（Record Lock）

单个行记录上的锁，总是会锁住索引记录，有以下几点问题：

1. 在不通过索引条件查询的时候，Innodb使用的是表锁，而不是行锁
2. 当表有多个索引的时候，不同的事务可以使用不同的索引锁定不同的行，另外，不论是使用主键索引、唯一索引或普通索引，InnoDB都会使用行锁来对数据加锁。
3. 即便在条件中使用了索引字段，但是否使用索引来检索数据是由MySQL通过判断不同执行计划的代价来决定的，如果MySQL认为全表扫描效率更高，比如对一些很小的表，它就不会使用索引，这种情况下InnoDB将使用表锁，而不是行锁。因此，在分析锁冲突时，别忘了检查SQL的执行计划，以确认是否真正使用了索引。

###### 间隙锁（GAP Lock）

间隙锁的作用是为了阻止多个事务将记录插入同一范围内，防止了幻读的出现

例如一个索引有10，11，13，20这四个值，那么该索引可能被Next-Key Locking的区间为：

(-无穷， 10]；

(10, 11];

(11, 13];

(13, 20];

(20, +无穷];

###### Next-Key Lock

是记录锁 + 间隙锁的；在查询的列是唯一索引的时候，Next-Key Lock 会降级为记录锁；因为在Repeatable Read 中Innodb就会使用间隙锁，所以Repeatable Read 就能够解决幻读

##### 缓冲池

因为在事务中提到了内存拷贝，所以看了一些关于Mysql缓存 缓冲池的知识

缓冲池是主存储器中的一个区域，用于在访问时缓存表和索引数据。缓冲池允许直接从内存处理常用数据，从而加快处理速度。在专用服务器上，通常会将最多80％的物理内存分配给缓冲池。

为了提高大容量读取操作的效率，缓冲池被分成可以容纳多行的页面。为了提高缓存管理的效率，缓冲池被实现为链接的页面列表; 使用LRU算法的变体，很少使用的数据在缓存中老化 。

**缓冲池LRU算法**

![](https://dev.mysql.com/doc/refman/5.7/en/images/innodb-buffer-pool-list.png)



Mysql缓冲池 分为两个部分 新字列表部分，旧字列表部分

其算法操作大致如下：

1. 3/8的缓冲池专用于旧子列表
2. 列表的中点是新子列表的尾部与旧子列表的头部相交的边界。
3. 当`InnoDB`将页面读入缓冲池时，它最初将其插入中点（旧子列表的头部）。可以读取页面，因为它是用户指定的操作（如SQL查询）所必需的，或者是由自动执行的[预读](https://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_read_ahead)操作的一部分 `InnoDB`。
4. 访问旧子列表中的页面使其 “ 年轻 ”，将其移动到缓冲池的头部（新子列表的头部）。如果因为需要而读取页面，则会立即进行第一次访问，并使页面变得年轻。如果由于预读而读取了页面，则第一次访问不会立即发生（并且在页面被逐出之前可能根本不会发生）。
5. 随着数据库的运行，在缓冲池的页面没有被访问的“ 年龄 ”通过向列表的尾部移动。新旧子列表中的页面随着其他页面的变化而变旧。旧子列表中的页面也会随着页面插入中点而老化。最终，仍然未使用的页面到达旧子列表的尾部并被逐出。

##### Innodb 页结构

InnoDB数据页由以下七个部分组成，如图所示： 

1. File Header（文件头）。

2. Page Header（页头）。

3. Infimun+Supremum Records。

4. User Records（用户记录，即行记录）。

5. Free Space（空闲空间）。

6. Page Directory（页目录）。

7. File Trailer（文件结尾信息）。

   ![](https://ws1.sinaimg.cn/large/a67bf22fgy1fu15d0d32kj20dx0f4ab8.jpg)

我这里主要想说三四点：

![](https://ws1.sinaimg.cn/large/a67bf22fgy1fu15edzuarj20kd0b9t9t.jpg)

非聚餐索引就是这种形式存储所有记录的；也就是说每个B+-Tree叶子结点都是一个页，存储了具体的记录；B+树索引本身并不能找到具体的一条记录，B+树索引能找到只是该记录所在的页。数据库把页载入内存，然后通过Page Directory再进行二叉查找。只不过二叉查找的时间复杂度很低，同时内存中的查找很快，因此通常我们忽略了这部分查找所用的时间。 

### 其它

##### 什么情况下会使用临时内存表

##### EXPLAIN 字段说明

- type

  1. system：表中只有一行数据或者是空表，且只能用于myisam和memory表。如果是Innodb引擎表，type列在这个情况通常都是all或者index

  2. const：使用唯一索引或者主键，返回记录一定是1行记录的等值where条件时，通常type是const。 

     ex：select * from student where student_id = 1;

  3. eq_ref：出现在要连接过个表的查询计划中，驱动表只返回一行数据，且这行数据是第二个表的主键或者唯一索引，且必须为not null，唯一索引和主键是多列时，只有所有的列都用作比较时才会出现eq_ref 

     

     ex：explain select * from book,student where student.id = book.id;

     ![](https://ws1.sinaimg.cn/large/a67bf22fgy1fu0b5mr61nj20wj026mx8.jpg)

  4. ref：不像eq_ref那样要求连接顺序，也没有主键和唯一索引的要求，只要使用相等条件检索时就可能出现，常见与辅助索引的等值查找。或者多列主键、唯一索引中，使用第一个列之外的列作为等值查找也会出现，总之，返回数据不唯一的等值查找就可能出现。 

     使用了辅助索引的返回列数不唯一的查询

     ex：select * from student where name="梅勇杰" and sex="男"

     ![](https://ws1.sinaimg.cn/large/a67bf22fgy1fu0bhiq21cj20y503hglq.jpg)二种情况：过滤条件都有索引；多列主键、唯一索引，使用第一个列之外的列作为等值查找；

  5. fulltext：全文索引检索，要注意，全文索引的优先级很高，若全文索引和普通索引同时存在时，mysql不管代价，优先选择使用全文索引 

  6. index_merge：表示查询使用了两个以上的索引，最后取交集或者并集，常见and ，or的条件使用了不同的索引，官方排序这个在ref_or_null之后，但是实际上由于要读取所有索引，性能可能大部分时间都不如range 

     ex：select * from student where name="梅勇杰" and sex="男"

     ![](https://ws1.sinaimg.cn/large/a67bf22fgy1fu0cdvfh6oj210102ft8w.jpg)

  7. unique_subquery：用于where中的in形式子查询，子查询返回不重复值唯一值 

  8. index_subquery：用于in形式子查询使用到了辅助索引或者in常数列表，子查询可能返回重复值，可以使用索引将子查询去重

  9. range：索引范围扫描，常见于使用 =, <>, >, >=, <, <=, IS NULL, <=>, BETWEEN, IN()或者like等运算符的查询中

  10. index：索引全表扫描，把索引从头到尾扫一遍，常见于使用索引列就可以处理不需要读取数据文件的查询、可以使用索引排序或者分组的查询

  11. all：这个就是全表扫描数据文件，然后再在server层进行过滤返回符合要求的记录

- partitions

  该列显示的为分区表命中的分区情况。 

- possible_keys

  查询可能使用到的索引都会在这里列出来 

- key

  查询真正使用到的索引，select_type为index_merge时，这里可能出现两个以上的索引，其他的select_type这里只会出现一个 

- key_len

  用于处理查询的索引长度，如果是单列索引，那就整个索引长度算进去，如果是多列索引，那么查询不一定都能使用到所有的列，具体使用到了多少个列的索引，这里就会计算进去，没有使用到的列，这里不会计算进去。留意下这个列的值，算一下你的多列索引总长度就知道有没有使用到所有的列了。要注意，mysql的ICP特性使用到的索引不会计入其中。另外，key_len只计算where条件用到的索引长度，而排序和分组就算用到了索引，也不会计算到key_len中。

- ref

  如果是使用的常数等值查询，这里会显示const，如果是连接查询，被驱动表的执行计划这里会显示驱动表的关联字段，如果是条件使用了表达式或者函数，或者条件列发生了内部隐式转换，这里可能显示为func 

- rows

  这里是执行计划中估算的扫描行数，不是精确值 ；即按照辅助索引查找到的应该去聚集索引查找的数据行

- filtered

  filtered = 最终结果的行数 / rows * 100% ； 即在Mysql服务器层对数据行过滤效果

- extra

  对于extra列，官网上有这样一段话：

  > If you want to make your queries as fast as possible, look out for Extra column values of Using filesort and Using temporary, or, in JSON-formatted EXPLAINoutput, for using_filesort and using_temporary_table properties equal to true.

  大概的意思就是说，如果你想要优化你的查询，那就要注意extra辅助信息中的using filesort和using temporary，这两项非常消耗性能，需要注意。

  这个列可以显示的信息非常多，有几十种，常用的有：
   A：distinct：在select部分使用了distinc关键字
   B：no tables used：不带from字句的查询或者From dual查询
   C：使用not in()形式子查询或not exists运算符的连接查询，这种叫做反连接。即，一般连接查询是先查询内表，再查询外表，反连接就是先查询外表，再查询内表。
   D：using filesort：排序时无法使用到索引时，就会出现这个。常见于order by和group by语句中
   E：using index：查询时不需要回表查询，直接通过索引就可以获取查询的数据。
   F：using join buffer（block nested loop），using join buffer（batched key accss）：5.6.x之后的版本优化关联查询的BNL，BKA特性。主要是减少内表的循环数量以及比较顺序地扫描查询。
   G：using sort_union，using_union，using intersect，using sort_intersection：
   using intersect：表示使用and的各个索引的条件时，该信息表示是从处理结果获取交集
   using union：表示使用or连接各个使用索引的条件时，该信息表示从处理结果获取并集
   using sort_union和using sort_intersection：与前面两个对应的类似，只是他们是出现在用and和or查询信息量大时，先查询主键，然后进行排序合并后，才能读取记录并返回。
   H：using temporary：表示使用了临时表存储中间结果。临时表可以是内存临时表和磁盘临时表，执行计划中看不出来，需要查看status变量，used_tmp_table，used_tmp_disk_table才能看出来。
   I：using where：表示存储引擎返回的记录并不是所有的都满足查询条件，需要在server层进行过滤。查询条件中分为限制条件和检查条件，5.6之前，存储引擎只能根据限制条件扫描数据并返回，然后server层根据检查条件进行过滤再返回真正符合查询的数据。5.6.x之后支持ICP特性，可以把检查条件也下推到存储引擎层，不符合检查条件和限制条件的数据，直接不读取，这样就大大减少了存储引擎扫描的记录数量。extra列显示using index condition
   J：firstmatch(tb_name)：5.6.x开始引入的优化子查询的新特性之一，常见于where字句含有in()类型的子查询。如果内表的数据量比较大，就可能出现这个
   K：loosescan(m..n)：5.6.x之后引入的优化子查询的新特性之一，在in()类型的子查询中，子查询返回的可能有重复记录时，就可能出现这个

  除了这些之外，还有很多查询数据字典库，执行计划过程中就发现不可能存在结果的一些提示信息

- filtered

  使用explain extended时会出现这个列，5.7之后的版本默认就有这个字段，不需要使用explain extended了。这个字段表示存储引擎返回的数据在server层过滤后，剩下多少满足查询的记录数量的比例，注意是百分比，不是具体记录数。 

#### 参考
explain字段说明：
[https://www.jianshu.com/p/73f2c8448722](https://www.jianshu.com/p/73f2c8448722)

[Mysql 一篇全面的博客](http://www.notedeep.com/note/38)





























