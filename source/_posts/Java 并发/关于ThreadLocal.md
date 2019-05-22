---
title: 关于ThreadLocal
date: 2018-10-30 20:11:29
tags: ThreadLocal
categories: java
---

#### 基本使用

#### ThreadLocal 是什么

ThreadLocalMap 是一个类似于HashMap的类，存储以ThreadLocal为键以想要存储的值为值的键值对，每个线程拥有自己的ThreadLocalMap

ThreadLocal 就是操作这个每个线程独有的ThreadLocalMap 的类

#### 从源码分析大致实现方式

##### set方法

```java
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }

```

1. 取得本线程自己的 ThreadLocalMap
2. 以ThreadLocal为键和以想要存储的值为值存进ThreadLocalMap

##### get 方法

```java
    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
```

1. 取得本线程的ThreadLocalMap
2. 以ThreadLocal 在ThreadLocalMap中取得存储的值

#### ThreadLocalMap 解决哈希冲突

ThreadLocalMap 没有采用链地址的方式来解决哈希冲突

```java
// ThreadLocalMap
private void set(ThreadLocal<?> key, Object value) {

        // We don't use a fast path as with get() because it is at
        // least as common to use set() to create new entries as
        // it is to replace existing ones, in which case, a fast
        // path would fail more often than not.

        Entry[] tab = table;
        int len = tab.length;
        int i = key.threadLocalHashCode & (len-1);

        for (Entry e = tab[i];
             e != null;
             e = tab[i = nextIndex(i, len)]) {
            ThreadLocal<?> k = e.get();

            if (k == key) {
                e.value = value;
                return;
            }

            if (k == null) {
                replaceStaleEntry(key, value, i);
                return;
            }
        }

        tab[i] = new Entry(key, value);
        int sz = ++size;
        if (!cleanSomeSlots(i, sz) && sz >= threshold)
            rehash();
    }
```
1. 根据threadLocalHashCode 每次加上固定的0x61c88647 大小得到下一个Index位置
2. 如果Index位置不为空且与key值相等，则替换值
3. 如果index位置不为空且与key值不相等，则寻找下一个空位置插入
4. 如果index位置为空那直接插入

#### ThreadLocal的内存泄漏

导致内存泄漏时因为下面的代码

```java
      static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
```

这个是ThreadLocalMap保存的键值对数组元素，我们可以看到ThreadLocal是弱引用，那么现在就存在一个问题，弱引用所引用的对象会在下一次垃圾回收回收掉；那么在我没有删除这个Entry，但是GC自动回收了ThreadLocal对象，造成了无法删除value

那么怎么解决呢？

在调用ThreadLocal的get()、set()可能会清除ThreadLocalMap中key为null的Entry对象，这样对应的value就没有GC Roots可达了，下次GC的时候就可以被回收，当然如果调用remove方法，肯定会删除对应的Entry对象。

如果使用ThreadLocal的set方法之后，没有显示的调用remove方法，就有可能发生内存泄露，所以养成良好的编程习惯十分重要，使用完ThreadLocal之后，记得调用remove方法。

```java
ThreadLocal<String> localName = new ThreadLocal();
try {
    localName.set("占小狼");
    // 其它业务逻辑
} finally {
    localName.remove();
}
```