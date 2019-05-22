---
title: 关于IO
date: 2019-03-10 15:06:37
tags: IO
---

FileChannel 的write和read方法都是线程安全的

#### BufferInputStream 为什么比较高效

以前一直有个误解，在BufferInputStream源码中存在一个byte数组，所以认为它高效的原因是不用每次都系统调用从外存获取数据；但是这里没有注意到磁盘缓存，磁盘缓存是为了缓和内存的速度和磁盘的速度根据局部性原理将读取数据的相邻数据都会一并读到磁盘缓存中；磁盘缓存存在于内核空间，所以每次去取数据都会存在用户空间和内核空间的切换；也就是说BufferInputStream 比较高效的原因是减少了用户空间和内核空间的切换从而节省了CPU资源

#### JAVA 中IO的方式

普通IO， FileChannel、MMAP(内存映射)

#### 为什么FileChannel 比普通IO要快

因为FileChannel采用了ByteBuffer这样的内存缓冲区，让我们可以精准的控制写盘的大小，其实就跟前面的BufferInputStream 一样，可以减少系统调用的次数，但是FileChannel可以根据系统自身情况调整ByteBuffer大小

#### 虚拟地址空间、逻辑地址、线性地址、物理地址

逻辑地址 如果以分页存储管理方式就是页号加页内偏移量

线性地址就是虚拟地址，这个地址存在的原因是内存交换技术，比如说32位的操作系统，每个进程可以拥有总的内存量为2^32 也就是4个G的内存，只不过是通过磁盘在逻辑上扩展物理内存进而达到好像该进程好像就拥有这么内存一样

逻辑地址通过地址转换机构得到的就是虚拟地址

物理地址就是真正物理意义上的物理内存上的内存单元地址

在没有使用虚拟存储器的机器上，虚拟地址被直接送到内存总线上，使具有相同地址的物理存储器被读写；而在使用了虚拟存储器的情况下，虚拟地址不是被直接送到内存地址总线上，而是送到存储器管理单元MMU，把虚拟地址映射为物理地址。

#### 直接IO

直接IO 存在原因PageCache也就是操作系统会将 I/O 的数据缓存在文件系统的页缓存（ page cache ）中，也就是说，数据会先被拷贝到操作系统内核的缓冲区中，然后才会从操作系统内核的缓冲区拷贝到应用程序的地址空间。DMA 方式可以将数据直接从磁盘读到页缓存中，或者将数据从页缓存直接写回到磁盘上，而不能直接在应用程序地址空间和磁盘之间进行数据传输，这样的话，数据在传输过程中需要在应用程序地址空间和页缓存之间进行多次数据拷贝操作，这些数据拷贝操作所带来的 CPU 以及内存开销是非常大的。

详细的可以见 [这篇IBM文章](https://www.ibm.com/developerworks/cn/linux/l-cn-directio/index.html)

但是需要注意的是JDK8是没有支持直接IO的类的

#### 直接内存

```java
ByteBuffer directBuffer = ByteBuffer.allocateDirect(1024);
```

通过上面这条语句分配的就是直接内存，一系列问题就出来了，什么是直接内存，它和JVM运行时数据区域有什么不同，它的好处是什么，下面就一一解答

直接内存就是一段 JVM运行时数据区域以外的内存，它不受JVM的管控，需要注意的是它依然属于用户内存

为什么要使用直接内存呢？ 这就牵扯到JAVA IO 读取数据的机制	

![](https://s2.ax1x.com/2019/03/11/A9oQ6H.png)

DMA 读取数据时先将数据读取到内核缓冲区当中，PageCache也就是在这个时候发生的；然后再将数据复制到堆外的一段内存当中，最后才将数据从堆外的直接内存复制到JVM 的堆内存当中，这里又会产生一个问题为什么不直接从内核缓冲区复制到JVM堆内存中，非要有一个中间堆外内存的复制过程？

这是因为虽然内核态下可以操作所有的内存，但是 它是通过物理地址的方式直接操作内存；所以这里就会产生一个问题，就是 JVM 对于大内存的分配都是直接在老年代内分配，JVM运行过程中很有可能会发生GC，而老年代大多数垃圾收集器又是用的标记整理算法，也就是会产生数据的移动，就会导致数据内存地址的变化，但是这个变化对于内核的复制是不可知的；所以从内核复制数据到的内存是不能够变化的，也就是这里的直接内存

下面是FileChannel.read方法调用的IOUtil.read方法，其中就可以看到这样的一个过程：

```java
static int read(FileDescriptor fd, ByteBuffer dst, long position,
                    boolean directIO, int alignment, NativeDispatcher nd)
        throws IOException
    {
        if (dst.isReadOnly())
            throw new IllegalArgumentException("Read-only buffer");
        // 如果是直接内存那么直接将数据读进直接内存中
        if (dst instanceof DirectBuffer)
            return readIntoNativeBuffer(fd, dst, position,
                    directIO, alignment, nd);

        // Substitute a native buffer
        ByteBuffer bb;
        int rem = dst.remaining();
        if (directIO) {
            Util.checkRemainingBufferSizeAligned(rem, alignment);
            bb = Util.getTemporaryAlignedDirectBuffer(rem,
                                                      alignment);
        } else {
        // 获取临时的直接内存
            bb = Util.getTemporaryDirectBuffer(rem);
        }
        try {
        // 将数据读进了临时直接内存中
            int n = readIntoNativeBuffer(fd, bb, position,
                    directIO, alignment,nd);
            bb.flip();
            // 再将直接内存的数据复制到堆内存中
            if (n > 0)
                dst.put(bb);
            return n;
        } finally {
            Util.offerFirstTemporaryDirectBuffer(bb);
        }
    }
```

至于为什么从直接内存到堆内存可以实现，可以参考来自知乎上的回答：

```txt
DirectByteBuffer 自身是一个Java对象，在Java堆中；而这个对象中有个long类型字段address，记录着一块调用 malloc() 申请到的native memory。

HotSpot VM里的GC除了CMS之外都是要移动对象的，是所谓“compacting GC”。

如果要把一个Java里的 byte[] 对象的引用传给native代码，让native代码直接访问数组的内容的话，就必须要保证native代码在访问的时候这个 byte[] 对象不能被移动，也就是要被“pin”（钉）住。

可惜HotSpot VM出于一些取舍而决定不实现单个对象层面的object pinning，要pin的话就得暂时禁用GC——也就等于把整个Java堆都给pin住。HotSpot VM对JNI的Critical系API就是这样实现的。这用起来就不那么顺手。

所以 Oracle/Sun JDK / OpenJDK 的这个地方就用了点绕弯的做法。它假设把 HeapByteBuffer 背后的 byte[] 里的内容拷贝一次是一个时间开销可以接受的操作，同时假设真正的I/O可能是一个很慢的操作。

于是它就先把 HeapByteBuffer 背后的 byte[] 的内容拷贝到一个 DirectByteBuffer 背后的native memory去，这个拷贝会涉及 sun.misc.Unsafe.copyMemory() 的调用，背后是类似 memcpy() 的实现。这个操作本质上是会在整个拷贝过程中暂时不允许发生GC的，虽然实现方式跟JNI的Critical系API不太一样。（具体来说是 Unsafe.copyMemory() 是HotSpot VM的一个intrinsic方法，中间没有safepoint所以GC无法发生）。

然后数据被拷贝到native memory之后就好办了，就去做真正的I/O，把 DirectByteBuffer 背后的native memory地址传给真正做I/O的函数。这边就不需要再去访问Java对象去读写要做I/O的数据了。
```

没有安全点，不会停下来GC

#### 直接内存的创建和回收机制

1. 创建

   关于直接内存的创建主要就涉及到它的构造器和reserveMemory方法

   大致流程：

   1. 调用reserveMemory 检查是否足够的直接内存分配

      reserveMemory 大致流程是这样的：

      1. 有足够内存直接返回，
      2. 尝试释放可能已经被回收的DirectBuffer 对象所关联的直接内存
      3. 如果 2 没有成功则 调用System.gc() 进行FullGC
      4. 循环等待9秒 等待FullGc完成
      5. 如果4没有成功那么抛出 直接内存溢出的Error

   2. 通过unsafe.allocateMemory申请直接内存，返回内存的首地址

   3. 构建Cleaner对象用于跟踪DirectByteBuffer对象的垃圾回收，以实现当DirectByteBuffer被垃圾回收时，堆外内存也会被释放

   ```java
       DirectByteBuffer(int cap) {                   // package-private
   
           super(-1, 0, cap, cap);
           boolean pa = VM.isDirectMemoryPageAligned();
           int ps = Bits.pageSize();
           long size = Math.max(1L, (long)cap + (pa ? ps : 0));
           Bits.reserveMemory(size, cap);
   
           long base = 0;
           try {
               base = unsafe.allocateMemory(size);
           } catch (OutOfMemoryError x) {
               Bits.unreserveMemory(size, cap);
               throw x;
           }
           unsafe.setMemory(base, size, (byte) 0);
           if (pa && (base % ps != 0)) {
               // Round up to page boundary
               address = base + ps - (base & (ps - 1));
           } else {
               address = base;
           }
           cleaner = Cleaner.create(this, new Deallocator(base, size, cap));
           att = null;
       }
   ```

   ```java
   static void reserveMemory(long size, int cap) {
   
       if (!memoryLimitSet && VM.initLevel() >= 1) {
           maxMemory = VM.maxDirectMemory();
           memoryLimitSet = true;
       }
   
       // optimist!
       if (tryReserveMemory(size, cap)) {
           return;
       }
   
       final JavaLangRefAccess jlra = SharedSecrets.getJavaLangRefAccess();
       boolean interrupted = false;
       try {
   
           // Retry allocation until success or there are no more
           // references (including Cleaners that might free direct
           // buffer memory) to process and allocation still fails.
           boolean refprocActive;
           do {
               try {
                   refprocActive = jlra.waitForReferenceProcessing();
               } catch (InterruptedException e) {
                   // Defer interrupts and keep trying.
                   interrupted = true;
                   refprocActive = true;
               }
               if (tryReserveMemory(size, cap)) {
                   return;
               }
           } while (refprocActive);
   
           // trigger VM's Reference processing
           System.gc();
   
           // A retry loop with exponential back-off delays.
           // Sometimes it would suffice to give up once reference
           // processing is complete.  But if there are many threads
           // competing for memory, this gives more opportunities for
           // any given thread to make progress.  In particular, this
           // seems to be enough for a stress test like
           // DirectBufferAllocTest to (usually) succeed, while
           // without it that test likely fails.  Since failure here
           // ends in OOME, there's no need to hurry.
           long sleepTime = 1;
           int sleeps = 0;
           while (true) {
               if (tryReserveMemory(size, cap)) {
                   return;
               }
               if (sleeps >= MAX_SLEEPS) {
                   break;
               }
               try {
                   if (!jlra.waitForReferenceProcessing()) {
                       Thread.sleep(sleepTime);
                       sleepTime <<= 1;
                       sleeps++;
                   }
               } catch (InterruptedException e) {
                   interrupted = true;
               }
           }
   
           // no luck
           throw new OutOfMemoryError("Direct buffer memory");
   
       } finally {
           if (interrupted) {
               // don't swallow interrupts
               Thread.currentThread().interrupt();
           }
       }
   }
   ```

2. 回收

   - 自动回收

     自动回收其实就是在分配内存的时候，如果发现直接内存不够的时候就会调用System.gc进行全局回收对象；但是问题就来了，对象的回收是怎么跟直接内存的回收挂钩的

     DirectByteBuffer对象在创建的时候关联了一个PhantomReference也就是下面这行代码：

     ```java
      cleaner = Cleaner.create(this, new Deallocator(base, size, cap));
     ```

     PhantomReference的作用是用来跟踪对象何时被回收的，它不能影响gc决策，但是gc过程中如果发现某个对象除了只有PhantomReference引用它之外，并没有其他的地方引用它了，那将会把这个引用放到java.lang.ref.Reference.pending队列里，在gc完毕的时候通知ReferenceHandler这个守护线程去执行一些后置处理，下面我会根据Reference的源码进一步说明

     Cleaner 继承于PhantomReference 并将其虚引用(其实就是 其子类Reference中的 referent的指向)指向了this即创建的DirectByteBuffer，就可以达到追踪DirectByteBuffer对象被回收的时机的效果

     如下面的代码所示，其实Cleaner 只是一个维护了Cleaner 双向链表的类，起到了保存回收其关联对象的回调方法即thunk属性的作用

     ```java
     public class Cleaner
         extends PhantomReference<Object>
     {
         private static final ReferenceQueue<Object> dummyQueue = new ReferenceQueue<>();
     
         private static Cleaner first = null;
     
         private Cleaner
             next = null,
             prev = null;
     
         private static synchronized Cleaner add(Cleaner cl) {
             if (first != null) {
                 cl.next = first;
                 first.prev = cl;
             }
             first = cl;
             return cl;
         }
     
         private static synchronized boolean remove(Cleaner cl) {
     
             // If already removed, do nothing
             if (cl.next == cl)
                 return false;
     
             // Update list
             if (first == cl) {
                 if (cl.next != null)
                     first = cl.next;
                 else
                     first = cl.prev;
             }
             if (cl.next != null)
                 cl.next.prev = cl.prev;
             if (cl.prev != null)
                 cl.prev.next = cl.next;
                 
             cl.next = cl;
             cl.prev = cl;
             return true;
     
         }
     
         private final Runnable thunk;
     
         private Cleaner(Object referent, Runnable thunk) {
             super(referent, dummyQueue);
             this.thunk = thunk;
         }
         
         public static Cleaner create(Object ob, Runnable thunk) {
             if (thunk == null)
                 return null;
             return add(new Cleaner(ob, thunk));
         }
     
         /**
          * Runs this cleaner, if it has not been run before.
          */
         public void clean() {
             if (!remove(this))
                 return;
             try {
                 thunk.run();
             } catch (final Throwable x) {
                 AccessController.doPrivileged(new PrivilegedAction<>() {
                         public Void run() {
                             if (System.err != null)
                                 new Error("Cleaner terminated abnormally", x)
                                     .printStackTrace();
                             System.exit(1);
                             return null;
                         }});
             }
         }
     }
     
      private static class ReferenceHandler extends Thread {
     
             private static void ensureClassInitialized(Class<?> clazz) {
                 try {
                     Class.forName(clazz.getName(), true, clazz.getClassLoader());
                 } catch (ClassNotFoundException e) {
                     throw (Error) new NoClassDefFoundError(e.getMessage()).initCause(e);
                 }
             }
     
             static {
                 // pre-load and initialize Cleaner class so that we don't
                 // get into trouble later in the run loop if there's
                 // memory shortage while loading/initializing it lazily.
                 ensureClassInitialized(Cleaner.class);
             }
     
             ReferenceHandler(ThreadGroup g, String name) {
                 super(g, null, name, 0, false);
             }
     
             public void run() {
                 while (true) {
                     processPendingReferences();
                 }
             }
         }
     ```

     下面就是比较重要的类了 Reference

     1. 其中有个静态代码块，这个类在rt.jar包下，所有是由BootStrapClassLoader加载的，也就是一开始就会执行这个静态代码块的内容，其中启动了一个ReferenceHandler的线程
     2. 其中详细的代码就不仔细讲了，如果有兴趣可以自己下去深究；它主要干的事情就是上面所说的gc过程中如果发现某个对象除了只有PhantomReference引用它之外，并没有其他的地方引用它了，那将会把这个引用放到java.lang.ref.Reference.pending队列里，在gc完毕的时候通知ReferenceHandler这个守护线程去执行一些后置处理，这里的后置处理就是Cleaner 的clean方法，clean方法又会调用当前对象保存的Runnable的run方法，即DirectByteBuffer 创建Cleaner 时传过去的Runnable，也就达到了追踪DirectByteBuffer销毁并做一定处理的效果，在DirectByteBuffer中所做的处理及传给Cleaner的Runnable 就是销毁自己创建的直接内存的操作

     ```java
     public abstract class Reference<T> {
     
        
         private T referent;         /* Treated specially by GC */
     
         ReferenceQueue<? super T> queue;
     
         Reference next;
         transient private Reference<T> discovered;  /* used by VM */
     
         static private class Lock { };
         private static Lock lock = new Lock();
         private static Reference pending = null;
         
         ...
     
         static {
             ThreadGroup tg = Thread.currentThread().getThreadGroup();
             for (ThreadGroup tgn = tg;
                  tgn != null;
                  tg = tgn, tgn = tg.getParent());
             Thread handler = new ReferenceHandler(tg, "Reference Handler");
             /* If there were a special system-only priority greater than
              * MAX_PRIORITY, it would be used here
              */
             handler.setPriority(Thread.MAX_PRIORITY);
             handler.setDaemon(true);
             handler.start();
         }
     
         public T get() {
             return this.referent;
         }
     
         public void clear() {
             this.referent = null;
         }
     
         public boolean isEnqueued() {
                 synchronized (this) {
                 return (this.queue != ReferenceQueue.NULL) && (this.next != null);
             }
         }
     
       
         public boolean enqueue() {
             return this.queue.enqueue(this);
         }
     
     
         /* -- Constructors -- */
     
         Reference(T referent) {
             this(referent, null);
         }
     
         Reference(T referent, ReferenceQueue<? super T> queue) {
             this.referent = referent;
             this.queue = (queue == null) ? ReferenceQueue.NULL : queue;
         }
     
     }
     
     
     private static void processPendingReferences() {
             waitForReferencePendingList();
             Reference<Object> pendingList;
             synchronized (processPendingLock) {
                 pendingList = getAndClearReferencePendingList();
                 processPendingActive = true;
             }
             while (pendingList != null) {
                 Reference<Object> ref = pendingList;
                 pendingList = ref.discovered;
                 ref.discovered = null;
                 //后置处理
                 if (ref instanceof Cleaner) {
                     ((Cleaner)ref).clean();
                     synchronized (processPendingLock) {
                         processPendingLock.notifyAll();
                     }
                 } else {
                     ReferenceQueue<? super Object> q = ref.queue;
                     if (q != ReferenceQueue.NULL) q.enqueue(ref);
                 }
             }
             synchronized (processPendingLock) {
                 processPendingActive = false;
                 processPendingLock.notifyAll();
             }
         }
     ```

     

#### 直接内存的应用场景

DirectBuffer 直接分配在JVM之外的物理内存，而不是 JVM 中的逻辑内存，需要往 Socket 或其他接口写的时候，不需要将数据从 JVM 复制到物理内存，直接输出即可。

缺点是当你需要对这些数据进行额外处理的时候，如编码，过滤等，数据还是会复制到 JVM，所以请确保你不需要对数据进行这些额外操作，只是从一个文件复制数据到另一个文件，一个Socket到另一个的时候才使用。

#### 内存映射文件

内存映射文件和之前说的 标准IO操作最大的不同之处就在于它虽然最终也是要从磁盘读取数据，但是它并不需要将数据读取到OS内核缓冲区，而是直接将进程的用户私有地址空间中的一 部分区域与文件对象建立起映射关系，就好像直接从内存中读、写文件一样，速度当然快了。为了说清楚这个，我们以 Linux操作系统为例子，看下图：

![](https://upload-images.jianshu.io/upload_images/5807849-9072d7c30c619a62.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/555/format/webp)

此图为 Linux 2.X 中的进程虚拟存储器，即进程的虚拟地址空间，如果你的机子是 32 位，那么就有 2^32 = 4G的虚拟地址空间，我们可以看到图中有一块区域： “Memory mapped region for shared libraries” ，这段区域就是在内存映射文件的时候将某一段的虚拟地址和文件对象的某一部分建立起映射关系，此时并没有拷贝数据到内存中去，而是当进程代码第一次引用这 段代码内的虚拟地址时，触发了缺页异常，这时候OS根据映射关系直接将文件的相关部分数据拷贝到进程的用户私有空间中去，当有操作第N页数据的时候重复这样的OS页面调度程序操作。注意啦，**原来内存映射文件的效率比标准IO高的重要原因就是因为少了把数据拷贝到OS内核缓冲区这一步**。

#### 为什么顺序读比随机读好

还是因为PageCache的原因，当顺序读的时候PageCache的命中率提高，更少的去磁盘读数据

#### FileChannel.transferTo

Java NIO中提供的FileChannel拥有transferTo和transferFrom两个方法，可直接把FileChannel中的数据拷贝到另外一个Channel，或者直接把另外一个Channel中的数据拷贝到FileChannel。该接口常被用于高效的网络/文件的数据传输和大文件拷贝。在操作系统支持的情况下，通过该方法传输数据并不需要将源数据从内核态拷贝到用户态，再从用户态拷贝到目标通道的内核态，同时也避免了两次用户态和内核态间的上下文切换，也即使用了“零拷贝”，所以其性能一般高于Java IO中提供的方法。

下面是简单的例子：

```
RandomAccessFile fromFile = new RandomAccessFile("fromFile.txt", "rw");
FileChannel fromChannel = fromFile.getChannel();
RandomAccessFile toFile = new RandomAccessFile("toFile.txt", "rw");
FileChannel toChannel = toFile.getChannel();
long position = 0;
long count = fromChannel.size();
toChannel.transferFrom(position, count, fromChannel);

```

#### 参考

[https://blog.csdn.net/coslay/article/details/44210993](https://blog.csdn.net/coslay/article/details/44210993)

[https://www.cnkirito.moe/file-io-best-practise/](https://www.cnkirito.moe/file-io-best-practise/)

