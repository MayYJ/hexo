---
title: HashMap源码分析
date: 2019-02-19 10:49:46
tags:
---

### 常见属性及其含义

```
   // 默认的初始化容量
   static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
   // 最大容量
    static final int MAXIMUM_CAPACITY = 1 << 30;
    // 默认的负载因子
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
    // 哈希桶里面的结点数大于该值就会 从链表转换为红黑树
    static final int TREEIFY_THRESHOLD = 8;
    // 哈希桶里面的结点数从较大值变为该值，就会从红黑树转换为链表
    static final int UNTREEIFY_THRESHOLD = 6;
    // 如果总容量小于该值但是哈希桶里面的结点数又大于threshold，就会扩容使结点分散
    static final int MIN_TREEIFY_CAPACITY = 64;
```

### 重要方法及其实现

#### putVal

**putVal**

1. 延迟初始化	

```java
//  在resize 方法里面完成初始化
if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
```

2. 定位：使用除留余数法完成hash桶的定位

```java
if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
```

对相同对象的判断方式

```java
if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
```

首先会判断hash值相同否，再判断equals方法是否相同；这样判断的原因是hashcode一样，不一定equals方法返回true

这里就涉及到一个原理 复写了equals方法是否需要复写hashcode

hashCode方法上面的注释有这样一句话，如果两个对象equals，那么它们的hashCode所返回的Integer应该是相等的，所以equals方法被重写意味着判断它们相等的方式产生了变化，也就需要根据新的条件来保证两个对象equals，hashCode方法返回的Integer相等

3. 红黑树化

```java
if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
```

当hash桶size > TREEIFY_THRESHOLD - 1 就会扩容

4. 扩容

将数组增大为原来的两倍，在转移过程中就会有二种情况：

1. 原hash表的hash桶只有一个元素，那么就直接添加到新hash表中
2. 当不只一个元素的时候，又有下面两种情况：
   1. 为红黑树，就按照红黑树将一棵树划分成两颗
   2. 为链表就按照新的hash表的容量做除留余数法，切割成两个链表添加进新的

#### resize

```java
if (oldCap > 0) {
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }
```

这里将capacity和threshold都扩大了两倍

```java
else {               // zero initial threshold signifies using defaults
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
```

这里是给putVal的延迟初始化用的，当table为NULL，那么就回使用resize进行初始化

下面就是具体的转移过程

### 一些常见问题

 #### 为什么hashMap的capacity要选取2的幂

因为这样(数组长度-1)正好相当于一个低位掩码，这样也正好对应了数组的长度，可以得到数组的一个下标

```
      10100101 11000100 00100101 	
&     00000000 00000000 00001111
--------------------------------
      00000000 00000000 00000101 //高位全部归零，只保留末四位
```

#### hash 方法原理

(h>>>16)因为Object.hashCode是返回int型的散列值也就是32位，这里右移了16位得到了hash值得高16位，前16位用0填充这样高位的信息被变相的保留了下来，再进行(h^(h>>>16))也就增加了低位的随机性下面使用的地方会使用 (n-1) & hash(key)就会随机得到一个Node数组的一个下标，随机性增强了那么碰撞的可能也就减少了

#### 为什么HashMap 非线程安全

- 这里只说明一个例子：现在有两个线程同时操作一个HashMap，他们现在同时取得了插入哈希表的同一个地址，现在第一个线程插入进去把next指针指向了它，但是第二个线程对此一无所知，依然进行了同样的操作，最后也就覆盖了第一个线程的操作。
- 还有个什么死循环看不怎么懂  .....

#### 在多线程中HashMap的死循环问题

在java1.8之后 JDK解决了多线程的死循环问题，所以下面贴的代码也是JDK8之前的

这是移动的逻辑

```java
void transfer(Entry[] newTable, boolean rehash) {
    int newCapacity = newTable.length;
    for (Entry<K,V> e : table) {
        while(null != e) {
            Entry<K,V> next = e.next;
            if (rehash) {
                e.hash = null == e.key ? 0 : hash(e.key);
            }
            int i = indexFor(e.hash, newCapacity);
            e.next = newTable[i];
            newTable[i] = e;
            e = next;
        }
    }
}
```

![](https://upload-images.jianshu.io/upload_images/2184951-67e51136429ece4c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/642/format/webp)

我们假设负载因子是1,所以当添加下一个元素时就会进行扩容操作

现在我们假设有A、B两个线程

1. A,B两个线程会分别创建自己的newTable

![](https://upload-images.jianshu.io/upload_images/2184951-c133d983c8c10681.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/688/format/webp)

2. 假设线程2运行到Entry<K,V> next = e.next 时间片用完了；这个时候e 指向 a 结点，next指向 b 结点；且在再hash的时候三个结点又同时在同一个hash桶

3. 在线程1 运行的时候就会按照从头结点加入的方式加入哈希桶中

   ![](https://upload-images.jianshu.io/upload_images/2184951-518ce8a7dc3a5532.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/378/format/webp)

   ![](https://upload-images.jianshu.io/upload_images/2184951-7559a35b8518c6a3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/564/format/webp)

   ![](https://upload-images.jianshu.io/upload_images/2184951-5666ecf1ef7c07cf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/706/format/webp)

   4. 这个时候线程1的时间片用完，线程2继续执行；就会将a结点加入自己新表中的哈希桶中，得到下图中的结果

      ![](https://upload-images.jianshu.io/upload_images/2184951-e2ac2c451982183b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/640/format/webp)

      5. 因为在上一个时间片已经赋值next为b；又会将哈希桶首位置变为b，b结点的next依然指向a

         ![](https://upload-images.jianshu.io/upload_images/2184951-cdede0c2ed25216c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/646/format/webp)

         6. 这个时候又取 b 结点的next 得到 a 结点，又将哈希桶首位置赋值 a 结点，且 a 结点的next 为 b结点;这样便形成了一个环

            ![](https://upload-images.jianshu.io/upload_images/2184951-ea3e9c3d3a407f01.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/670/format/webp)

            7. 这样不管最终那个线程 赋值了table，那么get 这个哈希桶的值的时候就会陷入死循环

#### 为什么线程安全的集合键值不可以为NULL而非线程安全的却可以

在例如HashMap非线程安全的集合 中可以设置键值为NULL 而不会产生到底是值本身是NULL还是哈希表中不存在这个键的歧义的原因是  在HashMap中我们可以使用ContainsKey来判断是否存在这个值 就避免了上述的歧义

而在线程安全的集合中	在 get 方法和 containsKey 方法之间可能存在其它线程删除键值对