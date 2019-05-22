---
title: AQS源码分析
date: 2019-01-28 14:34:20
tags: AQS
category: java
---

#### 概述

AQS 抽象队列同步器，是JAVA concurrent包锁机制实现基类，在JAVA1.6 虚拟机层面没有对Synchronized进行优化的时候  它的使用性能比synchronized 要好很多；JAVA 虚拟机层面一些锁优化 在AQS里面也有实现

AQS 实现了两种锁机制：共享锁和排他锁；ReentrantLock 相关类就是排他锁的实现类，CountDownLatch，CyclicBarrier就是共享锁的实现类

#### 数据结构

FiFO队列、双向链表

#### 重要属性

```java

    /** 等待队列的头结点
     * Head of the wait queue, lazily initialized.  Except for
     * initialization, it is modified only via method setHead.  Note:
     * If head exists, its waitStatus is guaranteed not to be
     * CANCELLED.
     */
    private transient volatile Node head;

    /**等待队列的尾结点
     * Tail of the wait queue, lazily initialized.  Modified only via
     * method enq to add new wait node.
     */
    private transient volatile Node tail;

    /**同步状态
     * The synchronization state.
     */
    private volatile int state;
```

#### FIFO队列的实现

```java
static final class Node {
        static final Node SHARED = new Node();
        static final Node EXCLUSIVE = null;
        static final int CANCELLED =  1; // 取消状态
        static final int SIGNAL    = -1; // 等待触发状态
        static final int CONDITION = -2; // 等待条件状态
        static final int PROPAGATE = -3; // 状态需要向后传播
        volatile int waitStatus;
        volatile Node prev;
        volatile Node next;
        volatile Thread thread;
        Node nextWaiter;
        ...
    }
```

#### 实现原理

##### 排它锁获取锁的过程

子类重写tryAcquire 和 tryRelease方法通过CAS指令修改状态变量state

```java
public final void acquire(int arg) {   
 if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))    
    selfInterrupt();
}
```

所以加锁操作就是两个操作：

1. 尝试直接修改state得到锁

2. 加入等待队列

   加入队列过程源码如下，大致总结一下过程

   1. 生成新的Node结点，并用CAS操作尝试插入到尾结点中；如果CAS操作没有成功，那么使用死循环强行使用CAS操作添加到队列中；在这里如果尾结点为空则赋值头结点为空，并让尾结点执行头结点
   2. node插入队尾后，并不会立马挂起，会进行自旋操作，判断在node插入过程中，刚刚运行的线程是否执行完成，如果pred = head说明当前结点是队列中第一个有效的结点，因此尝试tryAcquire获取锁
      1. 如果成功获取到锁，表明线程B已经执行完成，线程A不需要挂起。
      2. 如果获取失败，表示线程B还未完成，至少还未修改state值；继续下面的步骤
   3. 只有前一个结点pred的线程状态为SIGNAL时，当前结点的线程才能被挂起
      1. 如果pred的waitStatus == 0，则通过CAS指令修改waitStatus为Node.SIGNAL。
      2. 如果pred的waitStatus > 0，表明pred的线程状态CANCELLED，需从队列中删除。
      3. 如果pred的waitStatus为Node.SIGNAL，则通过LockSupport.park()方法把线程A挂起，并等待被唤醒，被唤醒后进入下一步
   4. 每次被唤醒都要进行中断检查，如果发现当前线程被中断，那么抛出InterruptedException并退出循环。从无限循环的代码可以看出，并不是被唤醒的线程一定能获得锁，必须调用tryAccquire重新竞争，因为锁是非公平的，有可能被新加入的线程获得，从而导致刚被唤醒的线程再次被阻塞，这个细节充分体现了“非公平”的精髓。

```java
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        enq(node);
        return node;
    }
```

```java
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)
        /*
         * This node has already set status asking a release
         * to signal it, so it can safely park.
         */
        return true;
    if (ws > 0) {
        /*
         * Predecessor was cancelled. Skip over predecessors and
         * indicate retry.
         */
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        /*
         * waitStatus must be 0 or PROPAGATE.  Indicate that we
         * need a signal, but don't park yet.  Caller will need to
         * retry to make sure it cannot acquire before parking.
         */
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```

##### 排它锁释放过程

1. 修改state和锁的偏向状态
2. 默认从队列下一个结点的线程unPark，但是就像上面说的它不一定能够获得运行的权力，因为还是要跟新加入的线程竞争

```java
    public final boolean release(int arg) {
        // 修改state和锁的偏向状态
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                // 默认去取下一个结点的线程来执行
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```

```java
private void unparkSuccessor(Node node) {
    /*
     * If status is negative (i.e., possibly needing signal) try
     * to clear in anticipation of signalling.  It is OK if this
     * fails or if status is changed by waiting thread.
     */
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    /*
     * Thread to unpark is held in successor, which is normally
     * just the next node.  But if cancelled or apparently null,
     * traverse backwards from tail to find the actual
     * non-cancelled successor.
     */
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread);
}
```

##### 弱一致性

在AQS 中存在弱一致性，维护一致性通常需要牺牲部分性能，为了进一步的提升性能，脑洞大开的神牛们想出了各种高性能的弱一致性模型。尽管模型允许了更多弱一致状态，但所有弱一致状态都在控制之下，不会出现一致性问题。

1. 在AQS 中比如说unparkSuccessor中 都是从tail开始往前遍历结点，这是因为AQS 中enq代码 没有对 前一个node的next属性维护一致性

   ```java
   private Node enq(final Node node) {
           for (;;) {
               Node t = tail;
               if (t == null) { // Must initialize
                   if (compareAndSetHead(new Node()))
                       tail = head;
               } else {
                   node.prev = t;
                   if (compareAndSetTail(t, node)) {
                       // 这里对竞态变量修改没有维护一致性，有可能在这个时候分配的时间片用完，下一个运行的线程并不会看到这个变化
                       t.next = node;
                       return t;
                   }
               }
           }
       }
   ```

   2. AQS 删除Canceled 节点过程中，也是弱一致性的

      ```java
      private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
              int ws = pred.waitStatus;
              if (ws == Node.SIGNAL)
                  /*
                   * This node has already set status asking a release
                   * to signal it, so it can safely park.
                   */
                  return true;
              if (ws > 0) {
                  /*
                   * Predecessor was cancelled. Skip over predecessors and
                   * indicate retry.
                   */
                  do {
                      // 这里不同线程可能看不到这里的修改
                      node.prev = pred = pred.prev;
                  } while (pred.waitStatus > 0);
                  pred.next = node;
              } else {
                  /*
                   * waitStatus must be 0 or PROPAGATE.  Indicate that we
                   * need a signal, but don't park yet.  Caller will need to
                   * retry to make sure it cannot acquire before parking.
                   */
                  compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
              }
              return false;
          }
      ```

      ##### 关于结点的状态

      ```java
      /** waitStatus value to indicate thread has cancelled */
              static final int CANCELLED =  1;
              /** waitStatus value to indicate successor's thread needs unparking */
              static final int SIGNAL    = -1;
              /** waitStatus value to indicate thread is waiting on condition */
              static final int CONDITION = -2;
              /**
               * waitStatus value to indicate the next acquireShared should
               * unconditionally propagate
               */
              static final int PROPAGATE = -3;
      ```

      1. CANCELLED 表示结点是取消状态，在获得了锁过后需要将其cancel
      2. SIGNAL 表示结点是待唤醒状态，表示结点所拥有线程被挂起需要被唤醒；在每次加入结点进入FIFO队列中的时候都需要先设置为这个状态
      3. CONDITION 表示结点等待条件触发的状态，注意这里的结点与上面的结点不一样，这里的结点是ConditionObject 里面维护的双向链表，相当于单独维护了一个等待池