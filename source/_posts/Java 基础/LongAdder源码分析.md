---
title: LongAdder源码分析
date: 2019-03-18 21:20:33
tags: LongAdder
category： java
---

#### LongAdder 概述

LongAdder是 JDK8 添加的高性能计数器，以前在并发环境下使用的计数器都是Atomic，Atomic是基于CAS操作，但是我们知道CAS操作有三个缺点(自行了解)；其中有一个就是可能因为高并发导致自旋太久；LongAdder使用了一种新的思想去解决计数，解决了上面的问题，下面就去探究是如何解决的

#### 继承结构

[![AnkJoT.md.png](https://s2.ax1x.com/2019/03/18/AnkJoT.md.png)](https://imgchr.com/i/AnkJoT)

从上面的继承结构图，可以看出Striped是其中的核心实现，下面会从源码角度去解析它

LongAdder、DoubleAdder和DoubleAccumulator、LongAccumulator的不同是前面两者是后面两者的简化实现；LongAdder、DoubleAdder只能完成加减操作而DoubleAccumulator、LongAccumulator可以自定义二元运算操作

#### 核心实现Striped64

介绍其中比较重要的字段

```java
 // 累计单元数组
 transient volatile Cell[] cells;
 // 累计器的基本值
 transient volatile long base;
 // 自旋标识符，在对cells进行初始化，或者后续扩容时，需要通过CAS操作把此标识设置为1
 transient volatile int cellsBusy;
```

介绍 longAccumulate 大致运行流程：

1. 首先获得线程私有的一个hash值，用于定位cells中的一个累计单元
2. 首先判断cells初始化没有，如果没有初始化就直接进行初始化
3. 如果正在进行初始化那么尝试直接更新base值
4. 如果初始化了，进行第一个 IF 里面的操作，我在源码里面写注释

```java
final void longAccumulate(long x, LongBinaryOperator fn,
                              boolean wasUncontended) {
        int h;
        if ((h = getProbe()) == 0) {
            ThreadLocalRandom.current(); // force initialization
            h = getProbe();
            wasUncontended = true;
        }
        boolean collide = false;                // True if last slot nonempty
        done: for (;;) {
            Cell[] cs; Cell c; int n; long v;
            if ((cs = cells) != null && (n = cs.length) > 0) {
                // 如果hash值定位的桶里面累计单元为空
                if ((c = cs[(n - 1) & h]) == null) {
                    if (cellsBusy == 0) {       // Try to attach new Cell
                        Cell r = new Cell(x);   // Optimistically create
                        // cellsBusy 其实就是一个扩容和添加Cell操作的锁
                        if (cellsBusy == 0 && casCellsBusy()) {
                            try {               // Recheck under lock
                                Cell[] rs; int m, j;
                                if ((rs = cells) != null &&
                                    (m = rs.length) > 0 &&
                                    rs[j = (m - 1) & h] == null) {
                                    rs[j] = r;
                                    break done;
                                }
                            } finally {
                                cellsBusy = 0;
                            }
                            continue;           // Slot is now non-empty
                        }
                    }
                    collide = false;
                }
                // 前面获取锁失败，也就是修改cellBusy失败，通过这里重试
                else if (!wasUncontended)       // CAS already known to fail
                    wasUncontended = true;      // Continue after rehash
                // 尝试直接修改hash值直接定位的累计单元，这里实现了扩容和修改的并行
                else if (c.cas(v = c.value,
                               (fn == null) ? v + x : fn.applyAsLong(v, x)))
                    break;
                // 这里限制了扩容，CPU能够并行的CAS操作的最大数量是它的核心数 
                else if (n >= NCPU || cells != cs)
                    collide = false;            // At max size or stale
                else if (!collide)
                    collide = true;
                // 获取锁进行扩容
                else if (cellsBusy == 0 && casCellsBusy()) {
                    try {
                        if (cells == cs)        // Expand table unless stale
                            cells = Arrays.copyOf(cs, n << 1);
                    } finally {
                        cellsBusy = 0;
                    }
                    collide = false;
                    continue;                   // Retry with expanded table
                }
                h = advanceProbe(h);
            }
            else if (cellsBusy == 0 && cells == cs && casCellsBusy()) {
                try {                           // Initialize table
                    if (cells == cs) {
                        Cell[] rs = new Cell[2];
                        rs[h & 1] = new Cell(x);
                        cells = rs;
                        break done;
                    }
                } finally {
                    cellsBusy = 0;
                }
            }
            // Fall back on using base
            else if (casBase(v = base,
                             (fn == null) ? v + x : fn.applyAsLong(v, x)))
                break done;
        }
    }
```

Cell[] 不能修改为 long[] 

因为在计数过程中，扩容和修改cell数组里面的单个计数单元是并行的；这里与ConcurrentHashMap 里面的扩容时类似的，在ConcurrentHashMap 中扩容的同时可以同时查找数据，但是不能够修改

#### LongAdder 实现

介绍一些重要方法

```java
// 首先尝试直接修改base的值，修改失败的话则通过Striped64 的longAccumulate实现并发计数
public void add(long x) {
        Cell[] as; long b, v; int m; Cell a;
        if ((as = cells) != null || !casBase(b = base, b + x)) {
            boolean uncontended = true;
            if (as == null || (m = as.length - 1) < 0 ||
                (a = as[getProbe() & m]) == null ||
                !(uncontended = a.cas(v = a.value, v + x)))
                longAccumulate(x, null, uncontended);
        }
    }

// 通过base 和 cell数组全部相加得到总的计数
public long sum() {
        Cell[] as = cells; Cell a;
        long sum = base;
        if (as != null) {
            for (int i = 0; i < as.length; ++i) {
                if ((a = as[i]) != null)
                    sum += a.value;
            }
        }
        return sum;
    }
```

