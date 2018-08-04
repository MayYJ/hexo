#### 关于ThreadLocal

1. 每个线程保存的是ThreadLocal.ThreadLocalMap

2. ThreadLocalMap跟HashMap类似，我们接下来就具体看看它的重要的一些源码

   - ```java
     
     //在ThreadLocal中保存的就是这个Entry数组，这个entry数组就是保存的我们的数据，Entry特殊的是它是一个弱引用，根据构造函数我们也知道它是让一个ThreadLocal 与一个我们要保存的数据进行绑定
     private Entry[] table;        
     
     static class Entry extends WeakReference<ThreadLocal<?>> {
                 /** The value associated with this ThreadLocal. */
                 Object value;
     
                 Entry(ThreadLocal<?> k, Object v) {
                     super(k);
                     value = v;
                 }
             }
     ```

   - 其它方面就跟HashMap非常类似，也有扩容机制，而且跟HashMap的扩容机制差不多