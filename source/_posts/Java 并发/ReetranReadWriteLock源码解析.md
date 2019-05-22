---
title: ReentranReadWriteLock源码解析
date: 2019-03-08 10:55:13
tags: ReentranReadWriteLock
---

因为ReentrantReadWriteLock是基于AQS实现的同步，所以在介绍这个类之前，首先要对AQS比较熟悉，所以涉及到它的知识点我不会加以解释

#### 大体框架

![](https://s2.ax1x.com/2019/03/09/ASGN0U.png)

- Sync

  ReentrantReadWriteLock 专门实现的AQS的同步器

- NonfairSync 、 FairSync

  公平锁和非公平锁的实现

- ReadLock

  读锁的实现，与Sync 是组合的关系

- WriteLock

  写锁的实现， 与Sync 是组合的关系

#### 重要方法解析

- tryAcquire

  运行流程：

  1. 如果readCount 或者 writeCount非零且owner 非本线程，获取锁失败
  2. 如果加锁数量太大(这个是受限于 16位的status 记录写线程数量)，获取锁失败
  3. 否则，当前线程可以因为是重入锁或者遵循锁策略 而获得锁，修改state、owner

```java
 protected final boolean tryAcquire(int acquires) {
            /*
             * Walkthrough:
             * 1. If read count nonzero or write count nonzero
             *    and owner is a different thread, fail.
             * 2. If count would saturate, fail. (This can only
             *    happen if count is already nonzero.)
             * 3. Otherwise, this thread is eligible for lock if
             *    it is either a reentrant acquire or
             *    queue policy allows it. If so, update state
             *    and set owner.
             */
            Thread current = Thread.currentThread();
            int c = getState();
            int w = exclusiveCount(c);
            if (c != 0) {
                // (Note: if c != 0 and w == 0 then shared count != 0)
                if (w == 0 || current != getExclusiveOwnerThread())
                    return false;
                if (w + exclusiveCount(acquires) > MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                // Reentrant acquire
                setState(c + acquires);
                return true;
            }
            if (writerShouldBlock() ||
                !compareAndSetState(c, c + acquires))
                return false;
            setExclusiveOwnerThread(current);
            return true;
        }
```

- tryAcquireShared

  运行流程:

  1. 如果其它线程拥有了写锁那么获取失败
  2. 否则，根据加锁策略加写锁，再CAS修改state，修改count
  3. 如果步骤2失败了，调用fullTryAcquireShared  死循环尝试修改

  ```java
          protected final int tryAcquireShared(int unused) {
              /*
               * Walkthrough:
               * 1. If write lock held by another thread, fail.
               * 2. Otherwise, this thread is eligible for
               *    lock wrt state, so ask if it should block
               *    because of queue policy. If not, try
               *    to grant by CASing state and updating count.
               *    Note that step does not check for reentrant
               *    acquires, which is postponed to full version
               *    to avoid having to check hold count in
               *    the more typical non-reentrant case.
               * 3. If step 2 fails either because thread
               *    apparently not eligible or CAS fails or count
               *    saturated, chain to version with full retry loop.
               */
              Thread current = Thread.currentThread();
              int c = getState();
              if (exclusiveCount(c) != 0 &&
                  getExclusiveOwnerThread() != current)
                  return -1;
              int r = sharedCount(c);
              if (!readerShouldBlock() &&
                  r < MAX_COUNT &&
                  compareAndSetState(c, c + SHARED_UNIT)) {
                  if (r == 0) {
                      firstReader = current;
                      firstReaderHoldCount = 1;
                  } else if (firstReader == current) {
                      firstReaderHoldCount++;
                  } else {
                      HoldCounter rh = cachedHoldCounter;
                      if (rh == null || rh.tid != getThreadId(current))
                          cachedHoldCounter = rh = readHolds.get();
                      else if (rh.count == 0)
                          readHolds.set(rh);
                      rh.count++;
                  }
                  return 1;
              }
              return fullTryAcquireShared(current);
          }
  ```

#### 如何保存读线程和写线程数量

```java

        static final int SHARED_SHIFT   = 16;
        static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
        static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
        static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;

        /** Returns the number of shared holds represented in count  */
        static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
        /** Returns the number of exclusive holds represented in count  */
        static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }

```

在ReentrantReadWriteLock 中用AQS的state变量包含了写锁的数量和读锁的数量

state 是一个int型变量，前16位用来表示读锁的数量，后16位用来表示写锁数量

#### 如何实现可重入

在写锁 里面依然跟ReentrantLock 一样使用的state变量，重入一次写锁数量就加1

读锁使用了ThreadLocal

```java
private transient ThreadLocalHoldCounter readHolds;

private transient HoldCounter cachedHoldCounter;

static final class ThreadLocalHoldCounter
            extends ThreadLocal<HoldCounter> {
            public HoldCounter initialValue() {
                return new HoldCounter();
            }
        }

static final class HoldCounter {
            int count = 0;
            // Use id, not reference, to avoid garbage retention
            final long tid = getThreadId(Thread.currentThread());
        }


```

读锁重入一次 count+1

#### 缓存的体现

```java
        private transient Thread firstReader = null;
        private transient int firstReaderHoldCount;

        private transient HoldCounter cachedHoldCounter;
```

这里根据应用场景，使用了两个缓存

1. 因为大多数ReentrantReadWriteLock 都只会被一个线程获取读锁，没有必要把它放到线程的ThreadLocalMap中，所以如果是第一次那么直接赋值firstReader、firstReaderHoldCount
2. 避免了释放锁的时候，从ThreadLocalMap 中取查询 holdCounter 

#### 为什么允许锁降级不能锁升级

锁降级：在拥有写锁的情况下去获取读锁

锁升级：在拥有读锁的情况下去获取写锁

锁升级会造成死锁，因为在线程拥有了读锁还没有释放的情况下，再去获取写锁那么就会造成当前线程阻塞等待读锁释放，也就造成了死锁的情况发生

#### 关于写饥饿问题

非公平模式下，在读多写少的情况下，会造成可能写线程等待读线程释放读锁后，但是因为又发生了获取了读并且修改了state变量，造成写线程获取写锁失败，进而导致修改不能及时反映

在ReentrantReadWriteLock  中启发函数中即 readerShouldBlock 中采用了如果当前等待队列中第一个等待的线程要获取写锁那么该获取读锁线程就不能去同写锁进行竞争插队而只能放到等待队列里面进行等待

所以这个可能造成一个问题，就是如果第一个读线程需要运行比较久的时间，而接下来是一个写线程等待，那么接下来的所有的读线程都只能等待，不能并发读数据

具体可见[https://zhuanlan.zhihu.com/p/34672421](https://zhuanlan.zhihu.com/p/34672421)

#### 应用场景

```
 * ReentrantReadWriteLocks can be used to improve concurrency in some
 * uses of some kinds of Collections. This is typically worthwhile
 * only when the collections are expected to be large, accessed by
 * more reader threads than writer threads, and entail operations with
 * overhead that outweighs synchronization overhead.
```

在collections 足够大且读线程比写线程多的情况下

#### 参考

[https://www.jianshu.com/p/6923c126e762](https://www.jianshu.com/p/6923c126e762)

