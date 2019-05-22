---
title: sychronize和wait/notify实现原理
date: 2019-02-19 14:42:44
tags: sychronized
category: Java并发
---

### 基本使用方法

1. 使用synchronized 修饰普通方法
2. 使用synchronized 修饰静态方法
3. 使用synchronized 修饰同步代码块

### Java 对象头

在HotSpot虚拟机中，Java对象由三部分组成：对象头，实例数据、对齐填充

对象头又由两部分组成：运行时数据（Mark Word），对象类型指针；

MarkWord在HotSpot中的实现在 markOop.hpp 文件中，一个对象在HotSpot中的实现在 oop.hpp 文件中

MarkWord 中的内容就如下图：

![](https://img-blog.csdn.net/20170603172215966?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvamF2YXplamlhbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

从上图也可以看出，偏向锁、轻量级锁、重量级锁的标志都在对象头中，所以他是锁实现的基本

### 字节码层面理解sychronized

1. 使用ACC_SYNCHRONIZED标记同步方法

```java
  public synchronized boolean test1();
    descriptor: ()Z
    flags: ACC_PUBLIC, ACC_SYNCHRONIZED
    Code:
      stack=1, locals=2, args_size=1
         0: iconst_1
         1: istore_1
         2: iconst_1
         3: ireturn
      LineNumberTable:
        line 4: 0
        line 5: 2
```

2. 使用monitorenter 和 monitorexit 同步

   ```java
    public boolean test1();
       descriptor: ()Z
       flags: ACC_PUBLIC
       Code:
         stack=2, locals=4, args_size=1
            0: aload_0
            1: dup
            2: astore_1
            3: monitorenter
            4: iconst_1
            5: istore_2
            6: iconst_1
            7: aload_1
            8: monitorexit
            9: ireturn
           10: astore_3
           11: aload_1
           12: monitorexit
           13: aload_3
           14: athrow
   ```

   不管是从哪种方式用哪种方式实现的同步，在虚拟中实现同步都是基于进入和退出管程(Monitor)对象来实现的

### 从虚拟机层面理解sychronized

#### Monitor 监控器

在MarkWord 中，如果是重量级锁的话，指向重量级锁的指针就是指向的一个Monitor对象；

其在HotSpot虚拟机源码实现在 ObjectMonitor.hpp文件

其数据结构如下：

```
ObjectMonitor() {
    _header       = NULL;
    _count        = 0; //记录个数
    _waiters      = 0,
    _recursions   = 0; // 用于实现可重入锁，下面会有介绍
    _object       = NULL;
    _owner        = NULL; // 用于标志拥有这个锁的线程，其实多线程下线程争抢不过就是这个字段；使用CAS将本线程赋值给这个字段，赋值成功就拥有了执行权限
    _WaitSet      = NULL; //处于wait状态的线程，会被加入到_WaitSet；等待池
    _WaitSetLock  = 0 ;
    _Responsible  = NULL ;
    _succ         = NULL ;
    _cxq          = NULL ; //处于等待锁block状态的线程，会被加入到该列表 没有明白与EntryList区别
    FreeNext      = NULL ;
    _EntryList    = NULL ; //处于等待锁block状态的线程，会被加入到该列表
    _SpinFreq     = 0 ;
    _SpinClock    = 0 ;
    OwnerIsThread = 0 ;
  }
```

#### 偏向锁实现

#### 轻量级锁实现

#### 重量级锁实现

上面三个标题见参考 占小狼 JVM源码分析之synchronized实现	

### 从虚拟机层面理解wait/notify

上面说过，ObjectMonitor对象中有两个队列：_WaitSet 和 _EntryList，用来保存ObjectWaiter对象列表；_owner指向获得ObjectMonitor对象的线程。

![](https://upload-images.jianshu.io/upload_images/2184951-9723bfce3c71c591.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/533/format/webp)

**_WaitSet ** ：处于wait状态的线程，会被加入到wait set；

**_EntryList**：处于等待锁block状态的线程，会被加入到entry set；  

1. wait 方法实现，下面我用语言描述过程，虚拟机实现参考 [占小狼 JVM源码分析之synchronized实现](https://www.jianshu.com/p/c5058b6fe8e5)

   1. 将当前线程封装成ObjectWaiter对象node
   2. 将上面创建的node 加入等待池中
   3. 释放当前对象的ObjectMonitor对象

2. notify方法实现

   取等待池中的第一个ObjectWaiter唤醒

### 参考

[占小狼 深入浅出synchronized](https://www.jianshu.com/p/19f861ab749e)

[占小狼 JVM源码分析之synchronized实现](https://www.jianshu.com/p/c5058b6fe8e5)

[占小狼 JVM源码分析之Object.wait/notify实现](https://www.jianshu.com/p/f4454164c017)

[深入理解Java并发之synchronized实现原理](https://blog.csdn.net/javazejian/article/details/72828483#synchronized%E5%BA%95%E5%B1%82%E8%AF%AD%E4%B9%89%E5%8E%9F%E7%90%86)