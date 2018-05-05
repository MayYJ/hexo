---
title: nio
date: 2018-05-05 17:41:16
tags: java
categories: java
---

##### java nio基本概念

nio就是new io，是相对于传统的io模型来说的；java nio是一种基于多路复用模型的同步非阻塞的io模型 。相对于传统就一个io流就需要一个线程来进行连接处理，nio的处理方式更加的节约资源，增加系统的吞吐量。

##### 同步和异步

主要站在程序的角度来看，防止共享资源的脏数据

- 同步：上一个程序没有完成之前，现在的程序只有轮询的方式得知上一个程序是否完成来决定现在的程序是否执行。
- 异步：用户程序发起操作后，便不管操作是否执行以及执行的内容；只有当操作完成之后，通过回调的方式来通知程序，然后接着去完成接下来的程序。

##### 阻塞与非阻塞

主要站在线程的角度

- 阻塞： 当一个线程发起操作后，具体执行线程时就一直等，等到实际操作完成后，线程才继续；
- 非阻塞：当一个线程发起操作后，具体执行的方法会分出独立的线程去执行操作，主线程会继续向下运行。

##### java nio的实现

![java-nio](http://www.godpan.me/media/images/2017/11/java-nio.png)

上面就是java nio的一种基本模型；一个线程对应一个selector，一个selector可以绑定多个Channel，一个Channel对应着一个Buffer。当然这只是通常的做法，一个Channel也可以对应多个Selector，一个Channel对应着多个Buffer。

###### selector

selector就是java nio实现多路复用的关键；在传统io中，一个socket我们必须用一个线程去管理；而在这里我们在io流和线程中间抽象出一个selector出来，selector就可以去管理多个io流连接从而实现多路链接。

###### channel

通道是java nio的第二个主要创新。它们既不是一个扩展也不是一项增强,而是全新、极好的 Java I/O 示例,提供与 I/O 服务的直接连接。Channel 用于在字节缓冲区和位于通道另一侧的实体(通常是一个文件或套接字)之间有效地传输数据。通道是一种途径,借助该途径,可以用最小的总开销来访问操作系统本身的 I/O 服务。

###### Buffer

一个Buffer对象是固定数量的数据的容器。其作用是一个存储器,或者分段运输区,在这里数据可被存储并在之后用于检索。缓冲区的工作与通道紧密联系。通道是 I/O 传输发生时通过的入口,而缓冲区是这些数据传输的来源或目标。对于离开缓冲区的传输,您想传递出去的数据被置于一个缓冲区,被传送到通道。

下面是一段怎么将ByteBuffer里面的数据转换为utf-8数据

```java

    private static String getBufferString(ByteBuffer buffer){
        Charset charset = null;
        CharsetDecoder decoder = null;
        CharBuffer charBuffer = null;
        try
        {
            charset = Charset.forName("UTF-8");
            decoder = charset.newDecoder();
            // charBuffer = decoder.decode(buffer);//用这个的话，只能输出来一次结果，第二次显示为空
            charBuffer = decoder.decode(buffer.asReadOnlyBuffer());
            return charBuffer.toString();
        }
        catch (Exception ex)
        {
            ex.printStackTrace();
            return "";
        }
    }
```



##### 在java nio中同步非阻塞的实现方式

因为java nio是在传统io中包装过来的，所以它的本质还是同步的，而它的非阻塞就是通过channel是实现的。在代码中我们通常通过 循环selector.select()来得到连接或者待读取的通道，这里都是同步的；当得到一个连接准备写入或者读取数据的时候也就是channel的write和read方法会异步的进行，也就是在执行write方法时在还没有进行数据写进buffer之前就返回了，read同理在没有读数据到Buffer的时候就已经返回，而是通过开启一个新的线程来完成写入和读取操作。

##### 简单的nio服务器代码实现

```java
import java.net.InetSocketAddress;
import java.net.ServerSocket;
import java.nio.ByteBuffer;
import java.nio.channels.SelectableChannel;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;
 
import javax.swing.text.html.HTMLDocument.Iterator;
 
/**
* Simple echo-back server which listens for incoming stream connections and
* echoes back whatever it reads. A single Selector object is used to listen to
* the server socket (to accept new connections) and all the active socket
* channels.
* @author zale (zalezone.cn)
*/
public class SelectSockets {
    public static int PORT_NUMBER = 1234;
    public static void main(String[] argv) throws Exception 
    {
        new SelectSockets().go(argv);
    }
    public void go(String[] argv) throws Exception 
    {
        int port = PORT_NUMBER;
        if (argv.length > 0) 
        { // 覆盖默认的监听端口
            port = Integer.parseInt(argv[0]);
        }
        System.out.println("Listening on port " + port);
        ServerSocketChannel serverChannel = ServerSocketChannel.open();// 打开一个未绑定的serversocketchannel
        ServerSocket serverSocket = serverChannel.socket();// 得到一个ServerSocket去和它绑定 
        Selector selector = Selector.open();// 创建一个Selector供下面使用
        serverSocket.bind(new InetSocketAddress(port));//设置server channel将会监听的端口
        serverChannel.configureBlocking(false);//设置非阻塞模式
        serverChannel.register(selector, SelectionKey.OP_ACCEPT);//将ServerSocketChannel注册到Selector
        while (true) 
        {
            // This may block for a long time. Upon returning, the
            // selected set contains keys of the ready channels.
            int n = selector.select();
            if (n == 0) 
            {
                continue; // nothing to do
            }           
            java.util.Iterator<SelectionKey> it = selector.selectedKeys().iterator();// Get an iterator over the set of selected keys
            //在被选择的set中遍历全部的key
            while (it.hasNext()) 
            {
                SelectionKey key = (SelectionKey) it.next();
                // 判断是否是一个连接到来
                if (key.isAcceptable()) 
                {
                    ServerSocketChannel server =(ServerSocketChannel) key.channel();
                    SocketChannel channel = server.accept();
                    registerChannel(selector, channel,SelectionKey.OP_READ);//注册读事件
                    sayHello(channel);//对连接进行处理
                }
                //判断这个channel上是否有数据要读
                if (key.isReadable()) 
                {
                    readDataFromSocket(key);
                }
                //从selected set中移除这个key，因为它已经被处理过了
                it.remove();
            }
        }
    }
       /**
    * Register the given channel with the given selector for the given
    * operations of interest
    */
    protected void registerChannel(Selector selector,SelectableChannel channel, int ops) throws Exception
    {
        if (channel == null) 
        {
            return; // 可能会发生
        }
        // 设置通道为非阻塞
        channel.configureBlocking(false);
        // 将通道注册到选择器上
        channel.register(selector, ops);
    }
    // ----------------------------------------------------------
    // Use the same byte buffer for all channels. A single thread is
    // servicing all the channels, so no danger of concurrent acccess.
    //对所有的通道使用相同的缓冲区。单线程为所有的通道进行服务，所以并发访问没有风险
    // 就是说这里因为只有一个selector 也就是
    private ByteBuffer buffer = ByteBuffer.allocateDirect(1024);
    * Sample data handler method for a channel with data ready to read.
    * 对于一个准备读入数据的通道的简单的数据处理方法
    * @param key
    *
    A SelectionKey object associated with a channel determined by
    the selector to be ready for reading. If the channel returns
    an EOF condition, it is closed here, which automatically
    invalidates the associated key. The selector will then
    de-register the channel on the next select call.
 
    一个选择器决定了和通道关联的SelectionKey object是准备读状态。如果通道返回EOF，通道将被关闭。
    并且会自动使相关的key失效，选择器然后会在下一次的select call时取消掉通道的注册

    protected void readDataFromSocket(SelectionKey key) throws Exception 
    {
        SocketChannel socketChannel = (SocketChannel) key.channel();
        int count;
        buffer.clear(); // 清空Buffer
        // Loop while data is available; channel is nonblocking
        //当可以读到数据时一直循环，通道为非阻塞
        while ((count = socketChannel.read(buffer)) > 0) 
        {
            buffer.flip(); // 将缓冲区置为可读
            // Send the data; don't assume it goes all at once
            //发送数据，不要期望能一次将数据发送完
            while (buffer.hasRemaining()) 
            {
                socketChannel.write(buffer);
            }
            // WARNING: the above loop is evil. Because
            // it's writing back to the same nonblocking
            // channel it read the data from, this code can
            // potentially spin in a busy loop. In real life
            // you'd do something more useful than this.
            //这里的循环是无意义的，具体按实际情况而定
            buffer.clear(); // Empty buffer
        }
        if (count < 0) 
                    {
            // Close channel on EOF, invalidates the key
            //读取结束后关闭通道，使key失效
            socketChannel.close();
        }
    }
    // ----------------------------------------------------------
    /**
    * Spew a greeting to the incoming client connection.
    *
    * @param channel
    *
    The newly connected SocketChannel to say hello to.
    */
    private void sayHello(SocketChannel channel) throws Exception 
    {
        buffer.clear();
        buffer.put("Hi there!\r\n".getBytes());
        buffer.flip();
        channel.write(buffer);
    }
}
```

##### 线程池的通道实现代码

```java
import java.io.IOException;
import java.nio.ByteBuffer;
import java.nio.channels.SelectionKey;
import java.nio.channels.SocketChannel;
import java.util.LinkedList;
import java.util.List;
 
/**
* Specialization of the SelectSockets class which uses a thread pool to service
* channels. The thread pool is an ad-hoc implementation quicky lashed togther
* in a few hours for demonstration purposes. It's definitely not production
* quality.
*
* @author Ron Hitchens (ron@ronsoft.com)
*/
public class SelectSocketsThreadPool extends SelectSockets 
{
    private static final int MAX_THREADS = 5;
    private ThreadPool pool = new ThreadPool(MAX_THREADS);
    // -------------------------------------------------------------
    public static void main(String[] argv) throws Exception 
    {
        new SelectSocketsThreadPool().go(argv);
    }
    // -------------------------------------------------------------
    /**
    * Sample data handler method for a channel with data ready to read. This
    * method is invoked from(被调用) the go( ) method in the parent class. This handler
    * delegates（委托） to a worker thread in a thread pool to service the channel,
    * then returns immediately.
    *
    * @param key
    *
    A SelectionKey object representing a channel determined by the
    *
    selector to be ready for reading. If the channel returns an
    *
    EOF condition, it is closed here, which automatically
    *
    invalidates the associated key. The selector will then
    *
    de-register the channel on the next select call.
    */
    protected void readDataFromSocket(SelectionKey key) throws Exception 
    {
        WorkerThread worker = pool.getWorker();
        if (worker == null) 
        {
            // No threads available. Do nothing. The selection
            // loop will keep calling this method until a
            // thread becomes available. This design could
            // be improved.
            return;
        }
        // Invoking this wakes up the worker thread, then returns
        worker.serviceChannel(key);
    }
    // ---------------------------------------------------------------
    /**
    * A very simple thread pool class. The pool size is set at construction
    * time and remains fixed. Threads are cycled through a FIFO idle queue.
    */
    private class ThreadPool
    {
        List idle = new LinkedList();
        ThreadPool(int poolSize) 
        {
            // Fill up the pool with worker threads
            for (int i = 0; i < poolSize; i++)
            {
                WorkerThread thread = new WorkerThread(this);
                // Set thread name for debugging. Start it.
                thread.setName("Worker" + (i + 1));
                thread.start();
                idle.add(thread);
            }
        }
        /**
        * Find an idle worker thread, if any. Could return null.
        */
        WorkerThread getWorker() 
        {
            WorkerThread worker = null;
            synchronized (idle) 
            {
                if (idle.size() > 0) 
                {
                    worker = (WorkerThread) idle.remove(0);
                }
            }
            return (worker);
        }
        /**
        * Called by the worker thread to return itself to the idle pool.
        */
        void returnWorker(WorkerThread worker) 
        {
            synchronized (idle) 
            {
                idle.add(worker);
            }
        }
    }
    /**
    * A worker thread class which can drain（排空） channels and echo-back（回显） the input.
    * Each instance is constructed with a reference（参考） to the owning thread pool
    * object. When started, the thread loops forever waiting to be awakened to
    * service the channel associated with a SelectionKey object. The worker is
    * tasked by calling its serviceChannel( ) method with a SelectionKey
    * object. The serviceChannel( ) method stores the key reference in the
    * thread object then calls notify( ) to wake it up. When the channel has
    * been drained, the worker thread returns itself to its parent pool.
    */
    private class WorkerThread extends Thread 
    {
        private ByteBuffer buffer = ByteBuffer.allocate(1024);
        private ThreadPool pool;
        private SelectionKey key;
        WorkerThread(ThreadPool pool) 
        {
            this.pool = pool;
        }
        // Loop forever waiting for work to do
        public synchronized void run() 
        {
            System.out.println(this.getName() + " is ready");
            while (true) 
            {
                try
                {
                    // Sleep and release object lock
                    //休眠并且释放掉对象锁
                    this.wait();
                } 
                catch (InterruptedException e) 
                {
                    e.printStackTrace();
                    // Clear interrupt status
                    this.interrupted();
                }
                if (key == null) 
                {
                    continue; // just in case
                }
                System.out.println(this.getName() + " has been awakened");
                try
                {
                    drainChannel(key);
                } 
                catch (Exception e) 
                {
                    System.out.println("Caught '" + e + "' closing channel");
                    // Close channel and nudge selector
                    try
                    {
                        key.channel().close();
                    } 
                    catch (IOException ex) 
                    {
                        ex.printStackTrace();
                    }
                    key.selector().wakeup();
                }
                key = null;
                // Done. Ready for more. Return to pool
                this.pool.returnWorker(this);
            }
        }
        /**
        * Called to initiate a unit of work by this worker thread on the
        * provided SelectionKey object. This method is synchronized, as is the
        * run( ) method, so only one key can be serviced at a given time.
        * Before waking the worker thread, and before returning to the main
        * selection loop, this key's interest set is updated to remove OP_READ.
        * This will cause the selector to ignore read-readiness for this
        * channel while the worker thread is servicing it.
        * 通过一个被提供SelectionKey对象的工作线程来初始化一个工作集合，这个方法是同步的，所以
        * 里面的run方法只有一个key能被服务在同一个时间，在唤醒工作线程和返回到主循环之前，这个key的
        * 感兴趣的集合被更新来删除OP_READ，这将会引起工作线程在提供服务的时候选择器会忽略读就绪的通道
        */
        synchronized void serviceChannel(SelectionKey key) 
        {
            this.key = key;
            key.interestOps(key.interestOps() & (~SelectionKey.OP_READ));
            this.notify(); // Awaken the thread
        }
        /**
        * The actual code which drains the channel associated with the given
        * key. This method assumes the key has been modified prior to
        * invocation to turn off selection interest in OP_READ. When this
        * method completes it re-enables OP_READ and calls wakeup( ) on the
        * selector so the selector will resume watching this channel.
        */
        void drainChannel(SelectionKey key) throws Exception 
        {
            SocketChannel channel = (SocketChannel) key.channel();
            int count;
            buffer.clear(); // 清空buffer
            // Loop while data is available; channel is nonblocking
            while ((count = channel.read(buffer)) > 0)
            {
                buffer.flip(); // make buffer readable
                // Send the data; may not go all at once
                while (buffer.hasRemaining()) 
                {
                    channel.write(buffer);
                }
                // WARNING: the above loop is evil.
                // See comments in superclass.
                buffer.clear(); // Empty buffer
            }
            if (count < 0) 
            {
                // Close channel on EOF; invalidates the key
                channel.close();
                return;
            }
            // Resume interest in OP_READ
            key.interestOps(key.interestOps() | SelectionKey.OP_READ);
            // Cycle the selector so this key is active again
            key.selector().wakeup();
        }
    }
}
```



##### 参考

http://www.importnew.com/22623.html

http://www.godpan.me/2017/11/05/java-nio.html

http://www.importnew.com/21341.html

http://outofmemory.cn/java/java-nio-note

https://blog.csdn.net/Sugar_Rainbow/article/details/77320741

https://wiki.jikexueyuan.com/project/java-nio/socketchannel.html