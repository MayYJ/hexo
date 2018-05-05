---
title: ArrayList源码解析
date: 2018-03-13 22:21:23
tags: ArrayList
categories: java
---

####Arrays.copyof

```
public static <T, U> T[] copyOf(U[] original, int newLength, Class<? extends T[]> newType) {
        T[] copy = newType == Object[].class ? new Object[newLength] : (Object[])Array.newInstance(newType.getComponentType(), newLength);
        System.arraycopy(original, 0, copy, 0, Math.min(original.length, newLength));
        return copy;
    }
```

函数解释：返回一个元素original数组一样的但是引用不一样的数组。

####System.arraycopy

```java
    public static native void arraycopy(Object src,  int  srcPos,
                                        Object dest, int destPos,
                                        int length);
```

函数解释：从指定的源数组复制数组，从指定位置，到目标数组的制定位置。源数组为src，目标数组为dest，复制的组件数量为length，从源数组复制的位置从下标srcPos到srcPos+length-1(源数组要复制的结尾的下标)，被复制到dest数组从下标destPos到destPos+length-1(目标数组要复制的结尾的下标)的位置。

####变量

```java
    private static final int DEFAULT_CAPACITY = 10; //默认的数组容量
    private static final Object[] EMPTY_ELEMENTDATA = new Object[0]; //临时实例化
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = new Object[0];
    transient Object[] elementData; //缓存
    private int size;//数组大小
    private static final int MAX_ARRAY_SIZE = 2147483639; //MAX_ARRAY_SIZE=Integer.MAX_VALNE-8;是因为虚拟机在数组数组类型数据中保留head word字段，其会占用空间。
```

####构造器

1. 无参构造函数

   ```
   public ArrayList() {
      this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
   }
   ```

   构造时是将空数组赋值给elementData ，但是在随后的第一个add元素的时候，会先新创建一个容量为10的初始数组。

2. 指定容量构造函数
   ```java
       public ArrayList(int initialCapacity) {
           if (initialCapacity > 0) {
               this.elementData = new Object[initialCapacity];
           } else {
               if (initialCapacity != 0) {
                   throw new IllegalArgumentException("Illegal Capacity: " + initialCapacity);
               }

               this.elementData = EMPTY_ELEMENTDATA;
           }

       }

   ```

3. 集合构造函数

   ````
       public ArrayList(Collection<? extends E> c) {
           this.elementData = c.toArray();
           if ((this.size = this.elementData.length) != 0) {
               if (this.elementData.getClass() != Object[].class) {
                   this.elementData = Arrays.copyOf(this.elementData, this.size, Object[].class);
               }
           } else {
               this.elementData = EMPTY_ELEMENTDATA;
           }

       }
   ````

####Method

##### add

```
    private void add(E e, Object[] elementData, int s) {
        if (s == elementData.length) {
            elementData = this.grow();
        }

        elementData[s] = e;
        this.size = s + 1;
    }
```

每一次添加元素的时候都会检查缓存数组的长度是否不够，不够就会加1

##### grow

```
    private Object[] grow() {
        return this.grow(this.size + 1);
    }
    
      private Object[] grow(int minCapacity) {
        return this.elementData =   Arrays.copyOf(this.elementData,this.newCapacity(minCapacity));
    }
```

##### newCapacity

```java
    private int newCapacity(int minCapacity) {
        int oldCapacity = this.elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1);// 
        if (newCapacity - minCapacity <= 0) {
            if (this.elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
                return Math.max(10, minCapacity);
            } else if (minCapacity < 0) {
                throw new OutOfMemoryError();
            } else {
                return minCapacity;
            }
        } else {
            return newCapacity - 2147483639 <= 0 ? newCapacity : hugeCapacity(minCapacity);
        }
    }
```

如果指定容量大于初始容量的1.5倍，且容量为空的话就初始化容量为10，不然就是指定容量的大小；所以这里当一个空的ArrayList添加一个元素后容量都会变为10.

##### remove

```
    public E remove(int index) {
        Objects.checkIndex(index, this.size);
        ++this.modCount;
        E oldValue = this.elementData(index);
        int numMoved = this.size - index - 1;
        if (numMoved > 0) {
            System.arraycopy(this.elementData, index + 1, this.elementData, index, numMoved);
        }

        this.elementData[--this.size] = null;
        return oldValue;
    }
```

把从要移除元素的下标位置开始每个元素向前移动一个位置，然后另最后一个元素为null；

##### addAll

```
    public boolean addAll(int index, Collection<? extends E> c) {
        this.rangeCheckForAdd(index);
        Object[] a = c.toArray();
        ++this.modCount;
        int numNew = a.length;
        if (numNew == 0) {
            return false;
        } else {
            Object[] elementData = this.elementData;
            int var10001 = this.elementData.length;
            int s = this.size;
            if (numNew > var10001 - this.size) {
                elementData = this.grow(s + numNew);
            }

            int numMoved = s - index;
            if (numMoved > 0) {
                System.arraycopy(elementData, index, elementData, index + numNew, numMoved);
            }

            System.arraycopy(a, 0, elementData, index, numNew);
            this.size = s + numNew;
            return true;
        }
    }
```

先扩大数组容量，然后从指定位置开始把所有元素向后面移动要添加元素数量的位置，然后再将要添加元素的从指定位置开始插入。

##### removeRange

```
    protected void removeRange(int fromIndex, int toIndex) {
        if (fromIndex > toIndex) {
            throw new IndexOutOfBoundsException(outOfBoundsMsg(fromIndex, toIndex));
        } else {
            ++this.modCount;
            this.shiftTailOverGap(this.elementData, fromIndex, toIndex);
        }
    }
```

从toIndex+1以后的所有元素移动到fromIndex的位置以后，然后再另length-1-（toIndex-fromIndex）位置后的所有元素为null

#### 一些内部类

##### ArrayListSpliterator

java8 并行迭代 Spliterator接口

Spliterator 是Java8 引入的新接口，顾名思义，**Spliterator可以理解为Iterator的Split版本，对于Java的流API，进行并行分割迭代计算，充分利用多核CPU的优势，并行计算具有极大的辅助作用**。在使用Iterator的时候，我们一般都是单线程地去顺序遍历集合的元素，但是使用Spliterator可以将集合元素分割成多份，使用多个线程 同时进行迭代，大大地提高了执行效率。

##### SubList

相当于ArrayList的一个子视图，所以对它的操作也会反应到ArrayList上，它的工作原理就是依赖于下面三个变量

```java
private final ArrayList<E> root; // 从root里面获得子list
private final int offset; //SubList的开始，root截断的起点
private int size;//subList的大小
```

##### Itr

ArrayList自定义实现的容器，实现了**fail-fast机制（在遍历过程中如果modCount与expectedModCount不相等，则抛出ConcurrentModificationException异常）**

#####  ListItr

继承自ArrayList<E>.Itr，扩展了ListIterator<E>的方法，能够在遍历过程中进行更多的操作。

#### 三种元素访问

1. 随机访问

```
String value = null;
int size = list.size();
for (int i=0; i<size; i++) {
    value = (String )list.get(i);        
}

```

此方法是直接在缓冲数组上的通过索引访问的，**速度最快**

2. foreach访问

```
String value = null;
for(String a : list){
    value = a;
}
```

foreach访问也比随机访问要慢，但是要快于迭代器的方式（foreach是一种**语法糖**，在编译期间需要进行**语法解析**，插入额外的辅助访问的代码，会有一定的消耗）

3. 迭代器访问

```
String value = null;
Iterator iter = list.iterator();
while (iter.hasNext()) {
    value = (String )iter.next();
}

```

**速度最慢**，由于要保存迭代器的状态，所以性能受到损耗

#### 一些需要注意的点

1. 底层通过System.arraycopy将原来ArrayList的**缓冲数组elementData拷贝给新的ArrayList的缓冲数组**，这里是一个深克隆，操作新的数组并不会改变原来的数组的状态。
2. **每一次影响集合结构的修改（包括增加、删除、扩容、移动元素位置，不包括修改set）**ArrayList的时候都要使得**modCount自增**，确保感知在使用**迭代器**和**进行序列化过程中**是否发生并发修改ArrayList的情况
3. 在子列表上的操作（如add、remove等）都会反映到原来的ArrayList上面（共用elementData），即子列表只是提供一种在原列表上的一种视图。