---
title: StampLock解析
date: 2019-03-09 18:49:31
tags: StampLock
---

#### 基本使用

[https://www.jianshu.com/p/481071ddafd3](https://www.jianshu.com/p/481071ddafd3)

#### 源码分析

[https://www.jianshu.com/p/bfd5d2321cc0](https://www.jianshu.com/p/bfd5d2321cc0)

#### 大概总结下它与ReentrantReadWriteLock的不同

1. StampedLock 更加高效，而他高价高效的原因是因为它在读模式、写模式的前提下引入了第三种模式乐观读模式；乐观读读模式下认为不会有其它写线程来修改当前线程要读取的数据，不用CAS操作去维护状态，使用了锁的版本号来识别是否有其它线程并发的修改了共享数据，所以使用它就要遵循一定的流程，具体流程见基本使用

2. StampedLock支持在三种模式中提供有条件的转换；这些方法的表现形式旨在帮助减少由于基于重试(retry-based)设计造成的代码膨胀。

3. StampedLock 不基于AQS，是不可重入的，所以在锁的内部不能调用其他尝试重复获取锁的方法,**StampedLocks是可序列化的，但是反序列化后变为初始的非锁定状态，所以在远程锁定中是不安全的**。

4. StampedLock  内部队列实现：

   ![](https://upload-images.jianshu.io/upload_images/6050820-c3187bc3f1c4c390.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/855/format/webp)

#### 应用场景

读多写少