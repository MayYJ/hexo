---
title: nio
date: 2018-05-05 17:41:16
tags: java
categories: java
---

#### IO模式

##### 异步IO

用户进程发起read操作之后，立刻就可以开始去做其它的事。而另一方面，从kernel的角度，当它受到一个asynchronous read之后，首先它会立刻返回，所以不会对用户进程产生任何block。然后，kernel会等待数据准备完成，然后将数据拷贝到用户内存，当这一切都完成之后，kernel会给用户进程发送一个signal，告诉它read操作完成了。 

![](https://ws1.sinaimg.cn/large/a67bf22fgy1fupj4idh0aj20fw0903zx.jpg)

##### 阻塞和非阻塞

- 阻塞

  在等待数据就绪和复制数据阶段均阻塞。 

  ![](https://ws1.sinaimg.cn/large/a67bf22fgy1fupj6q9kiqj20fc097t8n.jpg)

- 非阻塞

  在等待数据就绪阶段，如果数据未就绪 read 会立刻返回 error，不阻塞；用户需要轮询以确认数据就绪；当就绪后则复制数据，该过程阻塞。 

  ![](https://ws1.sinaimg.cn/large/a67bf22fgy1fupj8sj22xj20gr099aa4.jpg)

##### 多路复用

实现非阻塞

这个概念稍有不同，它是在执行 select() 的时候，同时阻塞多个 fd 然后等到监测到某些 fd 就绪时返回。此时进程两阶段均被阻塞，但等待数据就绪阶段由 select() 阻塞，复制数据阶段由 read() 阻塞。 

![](https://ws1.sinaimg.cn/large/a67bf22fgy1fupj7yfa1jj20gx092t8q.jpg)

在BIO模型中，我们要实现非阻塞，由于不能知道什么时候可以从内核缓冲区中取数据又不想去浪费CPU资源，那么我们只能创建一个新的线程，然后使用新的线程去做接下来的事件然后等到可以取数据的时候，我们再去取。但是创建线程也是很消耗资源，而且当线程多了后切换线程也是很耗CPU资源的。所以在单线程下的IO多路复用的优点就凸显数来了，没有线程切换，只有拼命的读、写、选择事件，如果再利用好多核心进行IO那么效率还会有更大的提升

##### Unix五种IO模型

- 阻塞IO
- 非阻塞IO
- IO复用（select、poll、epoll）
- 信号驱动IO
- 异步IO

##### Reactor和Proactor模式

其实java中Selector就是Reactor模式的实现，java中的AIO就是Proactor模式的实现；它们都要实现IO的多路复用，但是在事件分发者分发给事件处理者后（内核缓冲区数据准备好了）处理事件方式不一样；前者是同步的即在当前线程下处理IO任务（将内核缓冲区数据复制到用户空间），如果这个IO任务比较耗时就会比较浪费CPU资源；后者采用的方式的创建一个新的线程给事件处理者

#### NIO

##### NIO与传统IO的区别

1. NIO面向缓冲，而传统IO面向流；传统IO面向流意味着每次从流中读一个或多个字节，直至读取所有字节，它们没有被缓存在任何地方 
2. NIO是非阻塞的，而传统IO是阻塞的；在操作系统级别体现在当从IO设备接受数据的时候，在传统IO中当前线程需要等待设备控制器从IO设备中读取数据(虽然这个时候CPU与设备控制器是异步的)然后CPU再读取设备控制器中缓存的数据到内核内存里面；但是NIO会对前面这个过程进行异步，只有当IO设备里面的数据都到达了内核内存再通知操作系统(这里的通知的过程，我们完全可以参考后面对epoll的参考)

##### java nio基本概念

nio就是new io，是相对于传统的io模型来说的；java nio是一种基于多路复用模型的同步非阻塞的io模型 。相对于传统就一个io流就需要一个线程来进行连接处理，nio的处理方式更加的节约资源，增加系统的吞吐量。

##### java nio的实现

![java-nio](http://www.godpan.me/media/images/2017/11/java-nio.png)

上面就是java nio的一种基本模型；一个线程对应一个selector，一个selector可以绑定多个Channel，一个Channel对应着一个Buffer。当然这只是通常的做法，一个Channel也可以对应多个Selector，一个Channel对应着多个Buffer。

###### selector

selector就是java nio实现多路复用的关键；在传统io中，一个socket我们必须用一个线程去管理；而在这里我们在io流和线程中间抽象出一个selector出来，selector就可以去管理多个io流连接从而实现多路链接	。

- 创建selector

  ```
  Selector selector = Selector.open();
  ```

- 在selector上注册channel

  ```
  channel.configureBlocking(false);
  channel.register(selector, SelectionKey.OP_READ);
  ```

  这里的channel必须是非阻塞的，这里的第二个参数是表示channel对什么事件感兴趣，只有它感兴趣的事件selector才会分发给它，这里的事件有四种：

  1. Connect：一个channel成功连接到了其它服务器
  2. Accept： 一个ServerSocketChannel接受到了一个连接
  3. Read： 一个channel有数据等待读
  4. Write：一个channel准备好了写数据

  这四个事件对应SelectionKey四个常量：

  1. SelectionKey.OP_CONNECT
  2. SelectionKey.OP_ACCEPT
  3. SelectionKey.OP_READ
  4. SelectionKey.OP_WRITE

  需要理解SelectionKey.OP_READ、SelectionKey.OP_WRITE的使用，为什么一般情况下在读取了数据需要重新注册其感兴趣的事件为SelectionKey.OP_WRITE，因为在非阻塞的情况下，比如调用了read 就会立马返回，操作系统并行的去帮我们完成读数据这件事情，但是我们不知道什么时候完成，所以如果我们注册了SelectionKey.OP_WRITE 那么就在完成的时候得到通知

  如果想表示多个感兴趣的事件可以像如下：

  ```
  int interestSet = SelectionKey.OP_READ | SelectionKey.OP_WRITE;    
  ```

- SelectionKey

  其实根据源码我们知道SelectionKey绑定了一个channel和selector，就是想表示一个事件

  ```
      final SelChImpl channel;
      public final SelectorImpl selector;
      private int index;
      private volatile int interestOps;
      private int readyOps;
  ```

  1. 找到对应上面四种事件的SelectionKey

     ```
     selectionKey.isAcceptable();
     selectionKey.isConnectable();
     selectionKey.isReadable();
     selectionKey.isWritable();
     ```

  2. 通过SelectionKey产生selector和channel

     ```
     Channel  channel  = selectionKey.channel();
     Selector selector = selectionKey.selector();   
     ```

  3. 附加数据

     ```
     selectionKey.attach(theObject);
     
     Object attachedObj = selectionKey.attachment();
     ```

     又或者在注册的时候添加

     ```
     SelectionKey key = channel.register(selector, SelectionKey.OP_READ, theObject);
     ```

- 通过selector选择channel

  1. select() 阻塞到至少有一个channel已经准备好了你注册的事件
  2. int select(long timeout）和select()差不多，但是它最多阻塞timeout milliseconds 
  3. selectNow() 不阻塞，它会马上返回不管有没有channel准备好了

- 关于源码

  **selector 怎么过滤事件**

  ```java
      // 这是selector 监听得到的发生的感兴趣的事件
      protected Set<SelectionKey> selectedKeys = new HashSet();
      // 这是用户注册的通道感兴趣的事件
      protected HashSet<SelectionKey> keys = new HashSet();
      // 其实这两个都是上面两个的视图，read-only
      private Set<SelectionKey> publicKeys;
      private Set<SelectionKey> publicSelectedKeys;
      
      protected SelectorImpl(SelectorProvider var1) {
          super(var1);
          if (Util.atBugLevel("1.4")) {
              this.publicKeys = this.keys;
              this.publicSelectedKeys = this.selectedKeys;
          } else {
              this.publicKeys = Collections.unmodifiableSet(this.keys);
              this.publicSelectedKeys = Util.ungrowableSet(this.selectedKeys);
          }
      }
  ```

  **注册channel**

  ```java
   protected final SelectionKey register(AbstractSelectableChannel var1, int var2, Object var3) {
          if (!(var1 instanceof SelChImpl)) {
              throw new IllegalSelectorException();
          } else {
              SelectionKeyImpl var4 = new SelectionKeyImpl((SelChImpl)var1, this);
              var4.attach(var3);
              Set var5 = this.publicKeys;
              synchronized(this.publicKeys) {
                  this.implRegister(var4);
              }
  
              var4.interestOps(var2);
              return var4;
          }
      }
  ```

  下面的代码会调用selector的以上代码

  ```java
  serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
  ```

  大致完成了 实例化了一个SelectionKey并绑定了selector和channel，并且把此SelectionKey加入到了selector中的key中

###### channel

通道是java nio的第二个主要创新。它们既不是一个扩展也不是一项增强,而是全新、极好的 Java I/O 示例,提供与 I/O 服务的直接连接。Channel 用于在字节缓冲区和位于通道另一侧的实体(通常是一个文件或套接字)之间有效地传输数据。通道是一种途径,借助该途径,可以用最小的总开销来访问操作系统本身的 I/O 服务。

**channel有四种实现**

1. FileChannel：操作文件
2. DatagramChannel：在网络上读或者写数据通过UDP
3. SocketChannel：在网络上读或者写数据通过TCP
4. ServerSocketChannel：监听TCP的连接，就像一个web服务器；每一个连接到来都会有一个SocketChannel生成

**关于channel读数据**

因为在channel读数据的时候遇到了大大的坑 :angry:  NIO的API是真的有点难理解，真是搞飞机 :airplane: 所以在这里总结一下

处理读事件需要自己处理下列四种情况：

1. channel还有数据，需要继续读
2. channel中暂时没有数据，但channel还没有断开，这时读取到的数据个数为0，结束读，继续到select()处阻塞等待数据
3. 另外一端channel.close()关闭连接，这时候读channel返回的读取数是-1，表示已经到了到末尾，跟读文件一样；既然已经结束了，就把对应的SelectionKey给cancel掉，表示selector不再监听这个channel上的读事件；并且关闭连接，本端channel.close();
4. 另一端被强制关闭,也就是channel没有close()就被强制断开了,这时候本端会抛出一个IOException:你的主机中的软件中止了一个已建立的连接的异常，要处理这个异常

处理方法如下：

1. 在catch块里面关闭所有资源
2. 在再次循环的时候检查 selectionKey.isValid

```java
        while (flag) {
            if (selector.select() > 0) {
                Set<SelectionKey> selectionKeys = selector.selectedKeys();
                Iterator<SelectionKey> iterator = selectionKeys.iterator();

                while (iterator.hasNext()) {
                    SelectionKey selectionKey = iterator.next();
                    iterator.remove();
                    if (selectionKey.isValid() && selectionKey.isAcceptable()) {
                        SocketChannel client = ((ServerSocketChannel) selectionKey.channel()).accept();
                        InetSocketAddress socketAddress = (InetSocketAddress) client.getRemoteAddress();
                        System.out.println(socketAddress.getPort());
                        client.configureBlocking(false);
                        client.register(selector, SelectionKey.OP_READ);
                    }
                    if (selectionKey.isValid() && selectionKey.isReadable()) {
                        readData(selectionKey);
                    }
                    if (selectionKey.isValid() &&selectionKey.isWritable()) {

                    }
                    if (selectionKey.isValid() && selectionKey.isConnectable()) {
                        System.out.println("isConnectable == true");
                    }
                }
            }

        }
        
        
            private void readData(SelectionKey selectionKey) throws IOException {
        SocketChannel socketChannel = (SocketChannel) selectionKey.channel();
        try {
            InetSocketAddress socketAddress = (InetSocketAddress) socketChannel.getRemoteAddress();
            socketChannel.read(buffer);
            buffer.clear();
        } catch (IOException e) {
            selectionKey.cancel();
            socketChannel.socket().close();
            socketChannel.close();
            System.out.println("远程连接强行关闭了");
        }

//        socketChannel.register(selector, SelectionKey.OP_WRITE);
    }
```



**关于channel使用的技巧**

是否还记得，SelectionKey的四个事件，我们这里不讲SelectionKey.OP_CONNECT,其实我们控制channel就是通过register方法来修改它所感兴趣的事件来实现接下来这个channel所要去做的事件

是不是读着有点模糊，下面我结合代码来分析

```java
serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
```

上面这段代码，因为其作为服务端作用只有监听端口，接收连接，所以我们通过这句代码来控制它只接收 连接到来的事件，但是这句代码不能说明修改感兴趣的事件

```java
    private String readDataFromChannel(SelectionKey selectionKey) throws IOException {
        SocketChannel clientChannel = (SocketChannel) selectionKey.channel();
        StringBuilder readStr = new StringBuilder();
        int len;
        while ((len = clientChannel.read(buffer)) != 0 && len != -1) {
            buffer.flip();
            String str = new String(buffer.array(), 0, len);
            readStr.append(str);
            buffer.clear();
        }
        clientChannel.register(selector, SelectionKey.OP_WRITE);
        return readStr.toString();
    }
    
    private void sendData(SelectionKey selectionKey) throws IOException {
        SocketChannel clientChannel = (SocketChannel) selectionKey.channel();
        ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
        byte[] bytes = response.getBytes();
        byteBuffer.put(bytes);
        byteBuffer.flip();
        clientChannel.write(byteBuffer);
        clientChannel.register(selector, SelectionKey.OP_READ);
    }
```

上面的代码就比较典型，在接收到内容后，我们需要马上进行回复，那么就需要把该channel感兴趣的事件修改为SelectionKey.OP_WRITE；然后在回复完数据后，又要回到等待接收数据，就需要把该channel感兴趣的事件修改为SelectionKey.OP_READ

###### Buffer

一个Buffer对象是固定数量的数据的容器。其作用是一个存储器,或者分段运输区,在这里数据可被存储并在之后用于检索。缓冲区的工作与通道紧密联系。通道是 I/O 传输发生时通过的入口,而缓冲区是这些数据传输的来源或目标。对于离开缓冲区的传输,您想传递出去的数据被置于一个缓冲区,被传送到通道。

**buffer的基础用法**

1. 写数据到buffer
2. buffer.flip()
3. 从buffer读数据
4. buffer.clear() 或者 buffer.compact()

**capacity、position、limit**

position、limit的含义取决于buffer是出于写模式还是读模式，capacity的含义在这两种模式下的意义都是一样的

![](http://tutorials.jenkov.com/images/java-nio/buffers-modes.png)

在写模式下，position首先被置为0,limit被置为capacity，然后写一个数据类型的数据position就增加1，最大为capacity-1

当通过flip()变为读模式时，position变为0，limit变为写数据时写到最大的position位置，也就是只能从position读到limit

**Buffer类型**

- ByteBuffer
- MappedByteBuffer
- CharBuffer
- DoubleBuffer
- FloatBuffer
- IntBuffer
- LongBuffer
- ShortBuffer

**分配Buffer**

```
ByteBuffer buf = ByteBuffer.allocate(48);
CharBuffer buf = CharBuffer.allocate(1024);
```

**通过Buffer传输数据**

```
int bytesRead = inChannel.read(buf); //read into buffer.

buf.put(127);  
```

**flip()**

从写模式换到读模式，具体的代码实现就是，将position置为0，limit设置为上一次position的位置

下面是一段怎么将ByteBuffer里面的数据转换为utf-8数据

**clear() and compact()**

读完数据后可以通过clear()方法重新加入数据

当你从Buffer中没有读完数据，但是想要写数据，然后在读剩下的数据就用compact，具体实现就是把所有没有读的数据复制到buffer开始，position设置到没有读完的数据的右边

mark() and reset()

记住position

将position 设置到上一次mark记住position的位置

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

##### select、poll、epoll

这三者都是IO多路复用的机制。I/O多路复用就是通过一种机制，一个进程可以监视多个描述符，一旦某个描述符就绪（一般是读就绪或者写就绪），能够通知程序进行相应的读写操作。

- Linux的socket事件wakeup callback机制

  在介绍`select`、`poll`和`epoll`前，有必要说说Linux（2.6+）内核的事件wakeup callback机制，这是IO多路复用机制存在的本质。Linux通过socket睡眠队列来管理所有等待socket的某个事件的进程（Process），同时通过`wakeup`机制来异步唤醒整个睡眠队列上等待事件的Process，通知Process相关事件发生。

**select**

```
int select (int n, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
```

select 函数监视的文件描述符分3类，分别是writefds、readfds、和exceptfds。调用后select函数会阻塞，直到有描述副就绪（有数据 可读、可写、或者有except），或者超时（timeout指定等待时间，如果立即返回设为null即可），函数返回。当select函数返回后，可以 通过遍历fdset，来找到就绪的描述符。 

我大致讲一下其过程：

1. 当调用select 函数时，会将感兴趣的所有fd_set 从用户空间复制到内核空间
2. 当前线程会被加入到每个监控的Socket上的等待队列上面睡眠
3. 当Socket上有事件发生的时候修改复制进来的fd_set，唤醒等待的线程，等待的线程在继续执行遍历fd_set得到发生准备好的socket

select 主要有两个问题：

1.  可监控的fds太少 只能是1024
2. 每次都要遍历 fd_set,比较的耗时

**poll**

```
int poll (struct pollfd *fds, unsigned int nfds, int timeout);
```

不同与select使用三个位图来表示三个fdset的方式，poll使用一个 pollfd的指针实现。 

```
struct pollfd {
    int fd; /* file descriptor */
    short events; /* requested events to watch */
    short revents; /* returned events witnessed */
};
```

pollfd结构包含了要监视的event和发生的event，不再使用select“参数-值”传递的方式。同时，pollfd并没有最大数量限制（但是数量过大后性能也是会下降）。 和select函数一样，poll返回后，需要轮询pollfd来获取就绪的描述符。 

poll 只是把感兴趣事件的数据结构改了，支持监控更多的socket

**epoll**

```c
int epoll_create(int size)；
//创建一个epoll的句柄，size用来告诉内核这个监听的数目一共有多大，调用这个方法就会产生一个如下的结构体

struct eventpoll{
    ....
    /*红黑树的根节点，这颗树中存储着所有添加到epoll中的需要监控的事件*/
    struct rb_root  rbr;
    /*双链表中则存放着将要通过epoll_wait返回给用户的满足条件的事件*/
    struct list_head rdlist;
    ....
};
//我觉得rbr是在用户注册完需要监听的端口的所有事件后，系统在接受到一个请求后就会把此事件与这个红黑树rbr中的所有事件进行比较，如果有的话就会加入到rdlist，所以在用户不用遍历所有监听的端口而是只用遍历rdlist就可以得到所有可以读取的流

int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)
//创建需要监听的事件
epfd epoll的句柄id
op 操作即是要对fd即端口删除、增加还是修改event，所有op对应 添加EPOLL_CTL_ADD，删除EPOLL_CTL_DEL，修改EPOLL_CTL_MOD
fd 监听的端口对应的句柄
event 进行op操作的所有事件

int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
//从内核得到事件集合，返回需要处理的事件数目
```

以上我们可以得出epoll相对于select和poll的优点：

1. 没有注册描述符的限制，也就是没有监听端口的限制，也不会因为监听端口的增加而性能降低
2. 不用去遍历所有流得到能读或者写的流，epoll已经把哪个流产生了怎样的IO事件通知了我们，我们直接取出来即可

#### AIO

在我看来一下的两种方式其实都是通过一种方式来进行异步的：先是通过多路复用也就是Nio来实现非阻塞，然后再通过创建线程做其它事情的方式避免等待从内核缓冲区向用户空间复制数据

所以它们有一个共同点就是要先在用户空间创建buffer

**Future方式**

```java
   
    Path path =                                               Paths.get("/data/code/github/java_practice/src/main/resources/1log4j.properties");
    AsynchronousFileChannel channel = AsynchronousFileChannel.open(path);
    ByteBuffer buffer = ByteBuffer.allocate(1024);
    Future future = channel.read(buffer,0);
//        while (!future.isDone()){
//            System.out.println("I'm idle");
//        }
//我们可以在这里做其它事情
    Integer readNumber = future.get();

    buffer.flip();
    CharBuffer charBuffer = CharBuffer.allocate(1024);
    CharsetDecoder decoder = Charset.defaultCharset().newDecoder();
    decoder.decode(buffer,charBuffer,false);
    charBuffer.flip();
    String data = new String(charBuffer.array(),0, charBuffer.limit());
    System.out.println("read number:" + readNumber);
    System.out.println(data);
```

因为Future的本质就是直接返回而创建新的线程运算得到结果，运算结束后自己去取结果

**回调方式**

```java
    Path path = Paths.get("/data/code/github/java_practice/src/main/resources/1log4j.properties");
    AsynchronousFileChannel channel = AsynchronousFileChannel.open(path);
    ByteBuffer buffer = ByteBuffer.allocate(1024);
    channel.read(buffer, 0, buffer, new CompletionHandler() {
        @Override
        public void completed(Integer result, ByteBuffer attachment) {
            System.out.println(Thread.currentThread().getName() + " read success!");
        }

        @Override
        public void failed(Throwable exc, ByteBuffer attachment) {
            System.out.println("read error");
        }
    });

    while (true){
        System.out.println(Thread.currentThread().getName() + " sleep");
        Thread.sleep(1000);
    }
```

创建新的线程从内核取数据，所以completed方法也是在新线程上完成的

#### 从JAVA 源码分析NIO

##### 用NIO 实现一个简单的服务器

```java
 public void start() throws IOException {
        serverSocketChannel = ServerSocketChannel.open();
        ServerSocket serverSocket = serverSocketChannel.socket();
        serverSocket.bind(new InetSocketAddress(PORT));
        selector = Selector.open();
        serverSocketChannel.configureBlocking(false);
        serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);

        while (flag) {
            if (selector.select() > 0) {
                Set<SelectionKey> selectionKeys = selector.selectedKeys();
                Iterator<SelectionKey> iterator = selectionKeys.iterator();

                while (iterator.hasNext()) {
                    SelectionKey selectionKey = iterator.next();
                    iterator.remove();
                    if (selectionKey.isAcceptable()) {
                        SocketChannel client = ((ServerSocketChannel)selectionKey.channel()).accept();
                        client.configureBlocking(false);
                        client.register(selector, SelectionKey.OP_READ);
                    }
                    if (selectionKey.isReadable()) {
                        readData((SocketChannel) selectionKey.channel());
                    }
                    if (selectionKey.isConnectable()) {
                        System.out.println("isConnectable == true");
                    }

                }
            }

        }
    }
```

接下来我们就按照代码的逻辑思考底层是怎么实现的

##### Selector.open()

Selector 是Java NIO实现的核心，那么它是怎么怎么得到的，它是什么呢？

Selector 是通过SelectorProvider 得到的

```java
public static SelectorProvider provider() {
    synchronized (lock) {
        if (provider != null)
            return provider;
        return AccessController.doPrivileged(
            new PrivilegedAction<SelectorProvider>() {
                public SelectorProvider run() {
                        if (loadProviderFromProperty())
                            return provider;
                        if (loadProviderAsService())
                            return provider;
                        provider = sun.nio.ch.DefaultSelectorProvider.create();
                        return provider;
                    }
                });
    }
}
```

SelectorProvider 是一个桥连接实现的典型

本身SelectorProvider有自己的实现方式，在Linux和Windows下实现方式不同，然后它通过组合的方式成为比如说Selector的一个属性，然后Selector又有自己的不同实现，当然还有ServerSocketChannel

下面是Windows下Selector的一种实现

```java
    WindowsSelectorImpl(SelectorProvider sp) throws IOException {
        super(sp);
        // pollWrapper 是多路复用实现的关键；这个对象直接操作内存，将Socket句柄和感兴趣的事件保存保存下来
        pollWrapper = new PollArrayWrapper(INIT_CAP);
        // Pipe 是一种管道，既可以读也可以写
        // wakeupPipe 的实现是为了实现唤醒阻塞等待的Selector，下面会讲是怎么实现的
        wakeupPipe = Pipe.open();
        wakeupSourceFd = ((SelChImpl)wakeupPipe.source()).getFDVal();

        SinkChannelImpl sink = (SinkChannelImpl)wakeupPipe.sink();
        (sink.sc).socket().setTcpNoDelay(true);
        wakeupSinkFd = ((SelChImpl)sink).getFDVal();

        pollWrapper.addWakeupSocket(wakeupSourceFd, 0);
    }
```

##### pollWrapper

**pollWrapper**用Unsafe类申请一块物理内存pollfd，存放socket句柄fdVal和events，其中pollfd共8位，0-3位保存socket句柄，4-7位保存events。

![](https://upload-images.jianshu.io/upload_images/2184951-a074ce9d4031bb7b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/385/format/webp)

![](https://upload-images.jianshu.io/upload_images/2184951-327d066fab0e4047.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/488/format/web)

pollWrapper提供了fdVal和event数据的相应操作，如添加操作通过Unsafe的putInt和putShort实现。

##### serverSocketChannel.register()

这个方法其实 会调用Selector.register()

```java
   // SelectorImpl
    protected final SelectionKey register(AbstractSelectableChannel ch,
                                          int ops,
                                          Object attachment)
    {
        if (!(ch instanceof SelChImpl))
            throw new IllegalSelectorException();
        SelectionKeyImpl k = new SelectionKeyImpl((SelChImpl)ch, this);
        k.attach(attachment);
        synchronized (publicKeys) {
            implRegister(k);
        }
        // 添加感兴趣的事件
        k.interestOps(ops);
        return k;
    }

    protected void implRegister(SelectionKeyImpl ski) {
        synchronized (closeLock) {
            if (pollWrapper == null)
                throw new ClosedSelectorException();
            growIfNeeded();
            channelArray[totalChannels] = ski;
            ski.setIndex(totalChannels);
            fdMap.put(ski);
            keys.add(ski);
            // 添加Socket句柄
            pollWrapper.addEntry(totalChannels, ski);
            totalChannels++;
        }
    }
```

1. 生成SelectionKey并添加附件attachment
2. 如果当前channel的数量totalChannels等于SelectionKeyImpl数组大小，对SelectionKeyImpl数组和pollWrapper进行扩容操作。
3. 如果totalChannels % MAX_SELECTABLE_FDS == 0，则多开一个线程处理selector
4. 将SelectionKey 做上Index标记，标志Socket句柄在内存上的偏移量
5. 通过刚刚的Index属性添加 Socket句柄到对应的pollfd
6. 通过Index属性添加感兴趣的事件到对应的pollfd的event

##### doSelect()

```java
   protected int doSelect(long timeout) throws IOException {
        if (this.channelArray == null) {
            throw new ClosedSelectorException();
        } else {
            this.timeout = timeout;
            this.processDeregisterQueue();
            if (this.interruptTriggered) {
                this.resetWakeupSocket();
                return 0;
            } else {
                this.adjustThreadsCount();
                this.finishLock.reset();
                this.startLock.startThreads();

                try {
                    this.begin();

                    try {
                        this.subSelector.poll();
                    } catch (IOException var7) {
                        this.finishLock.setException(var7);
                    }

                    if (this.threads.size() > 0) {
                        this.finishLock.waitForHelperThreads();
                    }
                } finally {
                    this.end();
                }

                this.finishLock.checkForException();
                this.processDeregisterQueue();
                int var3 = this.updateSelectedKeys();
                this.resetWakeupSocket();
                return var3;
            }
        }
    }
```

其中 subSelector.poll() 是select的核心，由native函数poll0实现，readFds、writeFds 和exceptFds数组用来保存底层select的结果，数组的第一个位置都是存放发生事件的socket的总数，其余位置存放发生事件的socket句柄fd。

```java
private final int[] readFds = new int [MAX_SELECTABLE_FDS + 1];
private final int[] writeFds = new int [MAX_SELECTABLE_FDS + 1];
private final int[] exceptFds = new int [MAX_SELECTABLE_FDS + 1];
private int poll() throws IOException{ // poll for the main thread
    // 第一个参数是pollfd 数组的首地址，第二个参数就可以得到监控的所有channel,也就是Socket
     return poll0(pollWrapper.pollArrayAddress,
          Math.min(totalChannels, MAX_SELECTABLE_FDS),
             readFds, writeFds, exceptFds, timeout);
}
```

1.  如果之前没有发生事件，程序就阻塞在select处，当然不会一直阻塞，因为epoll在timeout时间内如果没有事件，也会返回；
2. 一旦有对应的事件发生，poll0方法就会返回；
3.  processDeregisterQueue方法会清理那些已经cancelled的SelectionKey；
4. updateSelectedKeys方法统计有事件发生的SelectionKey数量，并把符合条件发生事件的SelectionKey添加到selectedKeys哈希表中，并且把SelectionKey的readyOps修改为true

##### wakeUp()

```java
public Selector wakeup() {
    synchronized (interruptLock) {
        if (!interruptTriggered) {
            setWakeupSocket();
            interruptTriggered = true;
        }
    }
    return this;
}

// Sets Windows wakeup socket to a signaled state.
private void setWakeupSocket() {
   setWakeupSocket0(wakeupSinkFd);
}

private native void setWakeupSocket0(int wakeupSinkFd);
JNIEXPORT void JNICALL
Java_sun_nio_ch_WindowsSelectorImpl_setWakeupSocket0(JNIEnv *env, jclass this,
                                                jint scoutFd)
{
    /* Write one byte into the pipe */
    const char byte = 1;
    send(scoutFd, &byte, 1, 0);
}
```

可见wakeup()是通过pipe的write 端send(scoutFd, &byte, 1, 0)，发生一个字节1，来唤醒poll（）。所以在需要的时候就可以调用selector.wakeup()来唤醒selector。

#### epoll 实现原理

epoll是Linux下的一种IO多路复用技术，可以非常高效的处理数以百万计的socket句柄。

三个epoll相关的系统调用：

-  **int epoll_create(int size)**  
   epoll_create建立一个epoll对象。参数size是内核保证能够正确处理的最大句柄数，多于这个最大数时内核可不保证效果。
-  **int epoll_ctl(int epfd, int op, int fd, struct epoll_event event)**  
   epoll_ctl可以操作epoll_create创建的epoll，如将socket句柄加入到epoll中让其监控，或把epoll正在监控的某个socket句柄移出epoll。
-  **int epoll_wait(int epfd, struct epoll_event events,int maxevents, int timeout)**
   epoll_wait在调用时，在给定的timeout时间内，所监控的句柄中有事件发生时，就返回用户态的进程。

epoll内部实现大概如下：

1. epoll初始化时，会向内核注册一个文件系统，用于存储被监控的句柄文件，调用epoll_create时，会在这个文件系统中创建一个file节点。同时epoll会开辟自己的内核高速缓存区，以红黑树的结构保存句柄，以支持快速的查找、插入、删除。还会再建立一个list链表，用于存储准备就绪的事件。
2. 当执行epoll_ctl时，除了把socket句柄放到epoll文件系统里file对象对应的红黑树上之外，还会给内核中断处理程序注册一个回调函数，告诉内核，如果这个句柄的中断到了，就把它放到准备就绪list链表里。所以，当一个socket上有数据到了，内核在把网卡上的数据copy到内核中后，就把socket插入到就绪链表里。
3. 当epoll_wait调用时，仅仅观察就绪链表里有没有数据，如果有数据就返回，否则就sleep，超时时立刻返回。

#### 流和channel的不同

#### Reactor模型(反应堆模型)

- 什么是Reactor

下面是来自Wiki的解释：

The reactor design pattern is an event handling pattern for handling service requests delivered concurrently to a service handler by one or more inputs. The service handler then demultiplexes the incoming requests and dispatches them synchronously to the associated request handlers.

其中关键点为：

1. 事件驱动，即由某个事件的发生触发
2. 可以处理一个或多个输入源
3. 通过Service Handler同步的将输入事件（Event）采用多路复用分发给相应的Request Handler（多个）处理（注意这里有两个handler,一个ServiceHandler，一个Request Handler）

![](https://oscimg.oschina.net/oscnet/1f5c2e595e41559ac0c829a96a918c15f4e.jpg)

需要知道的是，Reactor模式是建立在Java NIO之上的，它只是一个很好的模式去连接事件和其相关联的事件处理器；它其实是一个流程化的过程，只是在JAVA 中用面向对象思想去建立这样的模型

- Reactor

  在反应堆模型中有三个角色，分别是：

  ```java
  1. Reactor 将IO事件分派给对应的handler
  2. Acceptor 处理客户端连接，并分派请求到处理器链中
  3. Handlers 执行非阻塞读/写 任务
  ```

  1. 单Reactor单线程模型

     ![](https://oscimg.oschina.net/oscnet/e9f813b5b08ac68021039ae5141c03f3cfc.jpg)

     概述一下其过程吧：

     - 客户端的连接会被分发器分发给accptor进行 SocketChannel的获取并将其处理器附加到SelectionKey上，以便后面事件发生的时候获取并处理
     - 当触发的事件发生比如说SocketChannel上有可以读或者可以写的事件发生，会通过分发器去获取SelectionKey上对应的处理器去执行

     下面是用代码模拟的模型：

     ```java
     
     /**
         * 等待事件到来，分发事件处理
         */
       class Reactor implements Runnable {
     ​
           private Reactor() throws Exception {
     ​
               SelectionKey sk =
                       serverSocket.register(selector,
                               SelectionKey.OP_ACCEPT);
               // attach Acceptor 处理新连接
               sk.attach(new Acceptor());
           }
     ​
           public void run() {
               try {
                   while (!Thread.interrupted()) {
                       selector.select();
                       Set selected = selector.selectedKeys();
                       Iterator it = selected.iterator();
                       while (it.hasNext()) {
                           it.remove();
                           //分发事件处理
                           dispatch((SelectionKey) (it.next()));
                       }
                   }
               } catch (IOException ex) {
                   //do something
               }
           }
     ​
           void dispatch(SelectionKey k) {
               // 若是连接事件获取是acceptor
               // 若是IO读写事件获取是handler
               Runnable runnable = (Runnable) (k.attachment());
               if (runnable != null) {
                   runnable.run();
               }
           }
     ​
       }
       /**
         * 连接事件就绪,处理连接事件
         */
       class Acceptor implements Runnable {
           @Override
           public void run() {
               try {
                   SocketChannel c = serverSocket.accept();
                   if (c != null) {// 注册读写
                       new Handler(c, selector);
                   }
               } catch (Exception e) {
     ​
               }
           }
       }
       /**
         * 处理读写业务逻辑
         */
       class Handler implements Runnable {
           public static final int READING = 0, WRITING = 1;
           int state;
           final SocketChannel socket;
           final SelectionKey sk;
     ​
           public Handler(SocketChannel socket, Selector sl) throws Exception {
               this.state = READING;
               this.socket = socket;
               sk = socket.register(selector, SelectionKey.OP_READ);
               sk.attach(this);
               socket.configureBlocking(false);
           }
     ​
           @Override
           public void run() {
               if (state == READING) {
                   read();
               } else if (state == WRITING) {
                   write();
               }
           }
     ​
           private void read() {
               process();
               //下一步处理写事件
               sk.interestOps(SelectionKey.OP_WRITE);
               this.state = WRITING;
           }
     ​
           private void write() {
               process();
               //下一步处理读事件
               sk.interestOps(SelectionKey.OP_READ);
               this.state = READING;
           }
     ​
           /**
             * task 业务处理
             */
           public void process() {
               //do something
           }
     ```

     因为整个过程都是在单线程下的，不能充分利用CPU的资源,且如果处理过程比较复杂会将整个过程阻塞，不能接受新的连接，所以衍生出了下面的模型

  2. 单Reactor多线程模型

     ![](https://oscimg.oschina.net/oscnet/1828d992e8821f9f093b6bf12c58732bb13.jpg)

     在这个模型下，其实其它都没有太多改变，就是添加了一个线程池去完成对读和写的过程的处理，

     其代码实现过程如下：

     ```java
     /**
         * 多线程处理读写业务逻辑
         */
       class MultiThreadHandler implements Runnable {
           public static final int READING = 0, WRITING = 1;
           int state;
           final SocketChannel socket;
           final SelectionKey sk;
     ​
           //多线程处理业务逻辑
           ExecutorService executorService = Executors.newFixedThreadPool(Runtime.getRuntime().availableProcessors());
     ​
     ​
           public MultiThreadHandler(SocketChannel socket, Selector sl) throws Exception {
               this.state = READING;
               this.socket = socket;
               sk = socket.register(selector, SelectionKey.OP_READ);
               sk.attach(this);
               socket.configureBlocking(false);
           }
     ​
           @Override
           public void run() {
               if (state == READING) {
                   read();
               } else if (state == WRITING) {
                   write();
               }
           }
     ​
           private void read() {
               //任务异步处理
               executorService.submit(() -> process());
     ​
               //下一步处理写事件
               sk.interestOps(SelectionKey.OP_WRITE);
               this.state = WRITING;
           }
     ​
           private void write() {
               //任务异步处理
               executorService.submit(() -> process());
     ​
               //下一步处理读事件
               sk.interestOps(SelectionKey.OP_READ);
               this.state = READING;
           }
     ​
           /**
             * task 业务处理
             */
           public void process() {
               //do IO ,task,queue something
           }
     }
     ```

  3. 多Reactor多线程模型

     ![](https://oscimg.oschina.net/oscnet/7ea7f4beb7b3c1d1c87d7b9e3bab8b6afb4.jpg)

     该模型后面的处理流程都与前面的模型一样，在单Reactor多线程模型下，连接的处理和去执行处理器是一个线性的过程，在该模型下，把连接的注册过程分离了出来

     mainReactor负责连接顺序的注册到subReactor上，subReactor管理自己的selector，对其上的SocketChannel进行监听

     ```javascript
     /**
         * 多work 连接事件Acceptor,处理连接事件
         */
       class MultiWorkThreadAcceptor implements Runnable {
     ​
           // cpu线程数相同多work线程
           int workCount =Runtime.getRuntime().availableProcessors();
           SubReactor[] workThreadHandlers = new SubReactor[workCount];
           volatile int nextHandler = 0;
     ​
           public MultiWorkThreadAcceptor() {
               this.init();
           }
     ​
           public void init() {
               nextHandler = 0;
               for (int i = 0; i < workThreadHandlers.length; i++) {
                   try {
                       workThreadHandlers[i] = new SubReactor();
                   } catch (Exception e) {
                   }
     ​
               }
           }
     ​
           @Override
           public void run() {
               try {
                   SocketChannel c = serverSocket.accept();
                   if (c != null) {// 注册读写
                       synchronized (c) {
                           // 顺序获取SubReactor，然后注册channel 
                           SubReactor work = workThreadHandlers[nextHandler];
                           work.registerChannel(c);
                           nextHandler++;
                           if (nextHandler >= workThreadHandlers.length) {
                               nextHandler = 0;
                           }
                       }
                   }
               } catch (Exception e) {
               }
           }
       }
       /**
         * 多work线程处理读写业务逻辑
         */
       class SubReactor implements Runnable {
           final Selector mySelector;
     ​
           //多线程处理业务逻辑
           int workCount =Runtime.getRuntime().availableProcessors();
           ExecutorService executorService = Executors.newFixedThreadPool(workCount);
     ​
     ​
           public SubReactor() throws Exception {
               // 每个SubReactor 一个selector 
               this.mySelector = SelectorProvider.provider().openSelector();
           }
     ​
           /**
             * 注册chanel
             *
             * @param sc
             * @throws Exception
             */
           public void registerChannel(SocketChannel sc) throws Exception {
               sc.register(mySelector, SelectionKey.OP_READ | SelectionKey.OP_CONNECT);
           }
     ​
           @Override
           public void run() {
               while (true) {
                   try {
                   //每个SubReactor 自己做事件分派处理读写事件
                       selector.select();
                       Set<SelectionKey> keys = selector.selectedKeys();
                       Iterator<SelectionKey> iterator = keys.iterator();
                       while (iterator.hasNext()) {
                           SelectionKey key = iterator.next();
                           iterator.remove();
                           if (key.isReadable()) {
                               read();
                           } else if (key.isWritable()) {
                               write();
                           }
                       }
     ​
                   } catch (Exception e) {
     ​
                   }
               }
           }
     ​
           private void read() {
               //任务异步处理
               executorService.submit(() -> process());
           }
     ​
           private void write() {
               //任务异步处理
               executorService.submit(() -> process());
           }
     ​
           /**
             * task 业务处理
             */
           public void process() {
               //do IO ,task,queue something
           }
       }
     
     ```

#### 参考

Linux io模式、select、poll、epoll

[https://segmentfault.com/a/1190000003063859](https://segmentfault.com/a/1190000003063859)

[https://blog.csdn.net/tianjing0805/article/details/76021440](https://blog.csdn.net/tianjing0805/article/details/76021440)

[这篇很好](https://juejin.im/entry/5b6058fde51d45348a2ffc65)

Aio

[https://juejin.im/entry/583ec2e3128fe1006bfa6c83](https://juejin.im/entry/583ec2e3128fe1006bfa6c83)

NIO

[http://tutorials.jenkov.com/java-nio/buffers.html](http://tutorials.jenkov.com/java-nio/buffers.html)

NIO 源码

[https://www.jianshu.com/p/0d497fe5484a](https://www.jianshu.com/p/0d497fe5484a)

[https://my.oschina.net/u/2337927/blog/523366](https://my.oschina.net/u/2337927/blog/523366)

反应堆

[https://blog.csdn.net/qq924862077/article/details/81026740](https://blog.csdn.net/qq924862077/article/details/81026740)

