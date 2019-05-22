---
title: ConcurrentHashMap源码分析
date: 2019-02-19 11:01:29
tags: ConcurrentHashMap
category: java
---

#### 数据结构

数组、链表、红黑树

#### 属性及其含义

```java
private static final int DEFAULT_CONCURRENCY_LEVEL = 16;
// 负载因子，即当前元素个数超过 length*LOAD_FACTOR 就会扩容
private static final float LOAD_FACTOR = 0.75f;
// 链表转为红黑树的阈值
static final int TREEIFY_THRESHOLD = 8;
// 红黑树转换为链表的阈值
static final int UNTREEIFY_THRESHOLD = 6;
// 链表能够转换为红黑树的最小的总的节点数
static final int MIN_TREEIFY_CAPACITY = 64;
// 并发转移结点时，每个线程每次转移最少的结点数
private static final int MIN_TRANSFER_STRIDE = 16;

// 哈希表
transient volatile Node<K,V>[] table;
// 临时哈希表，用于扩容时临时存放
private transient volatile Node<K,V>[] nextTable;
// 结点数量，与counterCells一起计算结点数量
private transient volatile long baseCount;
// -1 代表有线程在进行扩容
// -N 代表有N-1个线程在进行扩容
// 正数 表示table大小
private transient volatile int sizeCtl;
// 用于多个线程之间的扩容协作，当一个线程扩容后 transferIndex = transferIndex - stride
// 下一个线程就要从这个 被减后的transferIndex开始转移结点
private transient volatile int transferIndex;
private transient volatile CounterCell[] counterCells;
```

#### ConcurrentHashMap实现并发的方式

1. CAS
2. sychronized
3. volatile
4. 原子操作

#### 核心方法以及实现

##### putVal

1. 延迟初始化

   首先检查table初始化没有，如果没有那么就进行初始，如果在初始化时有其它线程访问，那么就让出时间片

   ```java
   if (tab == null || (n = tab.length) == 0)
                   tab = initTable()
   ```

2. 如果hash值的位置上为NULL,就直接通过CAS操作添加结点

   ```java
   else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                   if (casTabAt(tab, i, null,
                                new Node<K,V>(hash, key, value, null)))
                       break;                   // no lock when adding to empty bin
               }
   ```

3. 如果不能添加，并且发现在扩容，那么该线程就去帮助扩容；所以这里就涉及到了多线程扩容机制以及怎么使用FwordingNodeing 实现扩容的同时检索

   ```
   else if ((fh = f.hash) == MOVED)
                   tab = helpTransfer(tab, f);
   ```

   

4. 不是上面的情况就是要么hash值的位置上有结点要么是存在竞争CAS失败，那么就通过对哈希桶加锁添加

##### addCount 和 sumCount

有一些比较重要的字段需要讲解

**sizeCtl：** 用来控制table初始化和扩容操作

0或者正数也就是0.75n：表示没有进行初始化或者下次扩容大小

-1：代表table正在初始化

-N：代表有N-1个线程正在进行 扩容	

**baseCount**：在没有并发的时候，ConcurrentHashMap size加1直接加在这个字段上

**counterCells：**这是一个CounterCell的数组，在出现并发的不能利用CAS修改baseCount的时候，就会首先尝试随机取一个counterCell然后加1；如果上一个操作依然失败了那么就会调用addFullCount方法去实例化一个counterCell加1；

使用这个的目的是为了减轻对baseCount的负担；如果所有的添加都是在baseCount上操作，如果并发量大的话，必定在访问baseCount这里就会堆积大量的线程这样性能反而会差，但是如果把这些访问随机分配到CounterCells数组里，就会大大减少并发的堆积

**cellBusy**:这是修改counterCell的标志，在向counterCells添加元素的时候和counterCells扩容的时候其会被置为1

```java
// 从 putVal 传入的参数是 1， binCount，binCount 默认是0，只有 hash 冲突了才会大于 1.且他的大小是链表的长度（如果不是红黑数结构的话）。
private final void addCount(long x, int check) {
    CounterCell[] as; long b, s;
    // 如果计数盒子不是空 或者
    // 如果修改 baseCount 失败
    if ((as = counterCells) != null ||
        **!U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
        CounterCell a; long v; int m;
        boolean uncontended = true;
        // 如果计数盒子是空（尚未出现并发）
        // 如果随机取余一个数组位置为空 或者
        // 修改这个槽位的变量失败（出现并发了）
        // 执行 fullAddCount 方法。并结束
        if (as == null || (m = as.length - 1) < 0 ||
            // ThreadLocalRandom这个其实就是线程安全的随机数类
            (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
            !(uncontended =
              U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
            fullAddCount(x, uncontended);
            return;
        }
        if (check <= 1)
            return;
        s = sumCount();
    }
    // 如果需要检查,检查是否需要扩容，在 putVal 方法调用时，默认就是要检查的。
    if (check >= 0) {
        Node<K,V>[] tab, nt; int n, sc;
        // 如果map.size() 大于 sizeCtl（达到扩容阈值需要扩容） 且
        // table 不是空；且 table 的长度小于 1 << 30。（可以扩容）
        while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
               (n = tab.length) < MAXIMUM_CAPACITY) {
            // 根据 length 得到一个标识
            int rs = resizeStamp(n);
            // 如果正在扩容
            if (sc < 0) {
                // 如果 sc 的低 16 位不等于 标识符（校验异常 sizeCtl 变化了）
                // 如果 sc == 标识符 + 1 （扩容结束了，不再有线程进行扩容）（默认第一个线程设置 sc ==rs 左移 16 位 + 2，当第一个线程结束扩容了，就会将 sc 减一。这个时候，sc 就等于 rs + 1）
                // 如果 sc == 标识符 + 65535（帮助线程数已经达到最大）
                // 如果 nextTable == null（结束扩容了）
                // 如果 transferIndex <= 0 (转移状态变化了)
                // 结束循环 
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
                // 如果可以帮助扩容，那么将 sc 加 1. 表示多了一个线程在帮助扩容
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    // 扩容
                    transfer(tab, nt);
            }
            // 如果不在扩容，将 sc 更新：标识符左移 16 位 然后 + 2. 也就是变成一个负数。高 16 位是标识符，低 16 位初始是 2.
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                         (rs << RESIZE_STAMP_SHIFT) + 2))
                // 更新 sizeCtl 为负数后，开始扩容。
                transfer(tab, null);
            s = sumCount();
        }
    }
}
```

这个方法其实就是完成了两个动作：

1. ConcurrentHashMap size加1（下面是一个不断复杂的过程，有点类似于锁的膨胀过程）
   - 尝试直接修改baseCount，如果直接修改失败则执行下一步
   - 如果计数盒子为空即尚未出现过并发或者随机取一个计数盒子里面的位置为空或者尝试修改随机取得countCell失败，则执行下一步
   - fullAddCount 主要完成以下三件事情：如果CounterCells没有初始化则初始化；初始化了的就实例化CountterCell死循环添加进CounterCells；对CounterCells进行扩容
2. 检查是否需要扩容
   - 如果容量大于了阈值且不再扩容，那么赋值sizeCtl 开始扩容
   - 如果容量大于了阈值且正在扩容，那么帮助扩容
   - 这里是一个循环，进入的线程都会一直帮助扩容

```java
final long sumCount() {
        CounterCell[] as = counterCells; CounterCell a;
        long sum = baseCount;
        if (as != null) {
            for (int i = 0; i < as.length; ++i) {
                if ((a = as[i]) != null)
                    sum += a.value;
            }
        }
        return sum;
    }
```

在讲addCount方法第一步的时候其实已经囊括了讲解这里了

size 其实就是将baseCount加上因为并发冲突添加到CounterCell里面的数

##### 多线程扩容机制

###### 什么时候会扩容

在ConcurrentHashMap里面有两个地方存在扩容

- 在树化的时候，如果发现数组长度小于了`MIN_TREEIFY_CAPACITY`，默认是64 就会调用`tryPresize`方法把数组长度扩大到原来的两倍，并触发`transfer`方法，重新调整节点的位置。 
- 在添加结点的时候会调用`addCount`方法记录元素个数，并检查是否需要进行扩容，当数组元素个数达到阈值时，会触发`transfer`方法，重新调整节点的位置 

###### 怎么扩容的 (Transfer方法)

下面讲述大致过程：

1. 通过计算CPU核心数和Map数组的长度得到每个线程要帮助处理桶数；默认每个线程处理16个桶
2. 初始化nextTable为原table的两倍
3. 死循环开始转移；根据finishing变量来判断退出循环
   1. while循环有三个作用
      - 从大到小取每一个桶
      - 赋值 i 退出循环转移完成
      - 每个线程进行扩容的时候取得的transferIndex，得到转移的范围
   2. 转移完成的出口
   3. 如果oldTable 数组位置为NULL则赋为FowordNode表示该位置转移完成
   4. 如果oldTable 数组位置正在扩容就继续循环
   5. 具体的转移过程；这里会给每一个转移的哈希桶位置加锁，保证转移该位置上所有结点不会产生线程安全问题；转移数组位置上单个链表的时候会将其按照一定的算法分隔成两段然后插入到新的哈希表中；在转移完成后会将原哈希表该数组位置上赋值为ForwardingNode以便其它扩容线程看到表示该节点已经转移完成，也能够保证在扩容过程中依然能够支持哈希表的查找

```java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;
    // 计算每个线程分配的转移hash桶数
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; // subdivide range
    // 初始化nextTab
    if (nextTab == null) {            // initiating
        try {
            @SuppressWarnings("unchecked")
            // 扩容两倍
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
            nextTab = nt;
        } catch (Throwable ex) {      // try to cope with OOME
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        nextTable = nextTab;
        transferIndex = n;
    }
    int nextn = nextTab.length;
    // 实例化ForwardingNode hash桶转移完成的标志
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
    boolean advance = true;
    boolean finishing = false; // to ensure sweep before committing nextTab
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;
        while (advance) {
            int nextIndex, nextBound;
            // 向前推进 得到一个hash桶
            if (--i >= bound || finishing)
                advance = false;
            // 退出出口
            else if ((nextIndex = transferIndex) <= 0) {
                i = -1;
                advance = false;
            }
            // 每个线程从这里分配自己转移的hash桶的范围
            else if (U.compareAndSwapInt
                     (this, TRANSFERINDEX, nextIndex,
                      nextBound = (nextIndex > stride ?
                                   nextIndex - stride : 0))) {
                bound = nextBound;
                i = nextIndex - 1;
                advance = false;
            }
        }
        // 真正的出口，标志转移完成
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            if (finishing) {
                nextTable = null;
                table = nextTab;
                sizeCtl = (n << 1) - (n >>> 1);
                return;
            }
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;
                finishing = advance = true;
                i = n; // recheck before commit
            }
        }
        // 赋值原hash表NULL值hash桶为fwd，标志转移完成
        else if ((f = tabAt(tab, i)) == null)
            advance = casTabAt(tab, i, null, fwd);
        // 标志该hash桶转移完成
        else if ((fh = f.hash) == MOVED)
            advance = true; // already processed
        else {
            // 真正的转移过程
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    Node<K,V> ln, hn;
                    if (fh >= 0) {
                        int runBit = fh & n;
                        Node<K,V> lastRun = f;
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                            int b = p.hash & n;
                            if (b != runBit) {
                                runBit = b;
                                lastRun = p;
                            }
                        }
                        if (runBit == 0) {
                            ln = lastRun;
                            hn = null;
                        }
                        else {
                            hn = lastRun;
                            ln = null;
                        }
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            if ((ph & n) == 0)
                                ln = new Node<K,V>(ph, pk, pv, ln);
                            else
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                    else if (f instanceof TreeBin) {
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> lo = null, loTail = null;
                        TreeNode<K,V> hi = null, hiTail = null;
                        int lc = 0, hc = 0;
                        for (Node<K,V> e = t.first; e != null; e = e.next) {
                            int h = e.hash;
                            TreeNode<K,V> p = new TreeNode<K,V>
                                (h, e.key, e.val, null, null);
                            if ((h & n) == 0) {
                                if ((p.prev = loTail) == null)
                                    lo = p;
                                else
                                    loTail.next = p;
                                loTail = p;
                                ++lc;
                            }
                            else {
                                if ((p.prev = hiTail) == null)
                                    hi = p;
                                else
                                    hiTail.next = p;
                                hiTail = p;
                                ++hc;
                            }
                        }
                        ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                            (hc != 0) ? new TreeBin<K,V>(lo) : t;
                        hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                            (lc != 0) ? new TreeBin<K,V>(hi) : t;
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                }
            }
        }
    }
}
```

大致总结上面的终点就是：

1. 数组扩大两倍
2. 通过CPU数量计算每个线程转移数量
3. 每个线程通过transferIndex计算自己转移的范围
4. 具体转移过程中，会将原链表或者红黑树分隔，并且转移过程中转移完成的位置会用FowrdingNoding表示转移完成

##### ConcurrentHashMap保证数组可见性

在JAVA 中volatile字段修饰数组，只能保证对数组引用修改的可见性，不能保证对数组元素存取的可见性；在ConcurrentHashMap中，tabAt 方法中使用getVolatileValue保证了对取的可见性，使用CAS操作保证了对存的可见性

```java
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
        return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
}
```

##### 怎么保证GET方法 不用加锁

1.  ConcurrentHashMap 使用volatile、CAS、sychronized、getVolatileValue 保证了所有操作的可见性
2.  使用FwordingNode 保证了在扩容的同时依然能够查询





