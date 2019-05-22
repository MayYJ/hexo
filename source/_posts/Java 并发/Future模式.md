---
title: Future模式
date: 2019-02-12 19:36:06
tags:  Future模式
category: java
---

#### 并发模式: Future模式

Future模式有其基本结构，如下：

| 对象       | 作用                                                       |
| ---------- | ---------------------------------------------------------- |
| Client     | 取数据，立即返回FutureData，并新建线程构建RealData         |
| Data       | 接口，FutureData，RealData都实现于它                       |
| FutureData | Future数据，是RealData的封装，用于立即返回，是一个虚拟数据 |
| RealData   | 真实的数据，构造比较慢                                     |

以下就是上述对象的基本模型：

```java
public class Client{
   // 用于请求，立即返回FutureData，并在其中新建线程构造RealData然后赋值给FutureData
    public Data request(T t){
        ...
    }
}

public class Data{
    public T getResult();
}

public class FutureData implements Data{
    private RealData realData;
    private boolean ready;
    // 
    public synchronized T getResult(){
        if(!ready)
        {
            try {
                wait();
            }catch(InterruptException e) {
                ....
            }
        }
        return realData.result;
    }
    
    public synchronized void setResult(RealData realData) {
        if(ready) {
            return;
        }
        this.realData = realData;
        ready = true;
        notifyAll();
    }
}
```

#### JDK 中的Future 模式

在JDK 中是将上述Future模式的基本对象全部封装在了FutureTask中了；

其中run方法就实现了构造真实对象，并且改变状态赋值结果

```java
    public void run() {
        if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
            return;
        try {
            Callable<V> c = callable;
            if (c != null && state == NEW) {
                V result;
                boolean ran;
                try {
                    result = c.call();
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    setException(ex);
                }
                if (ran)
                    set(result);
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts
            int s = state;
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
    }
```

get方法就类似于FutureData的getResult

```java
public V get() throws InterruptedException, ExecutionException {
    int s = state;
    if (s <= COMPLETING)
        s = awaitDone(false, 0L);
    return report(s);
}
```

一般我们就将线程池作为Client，将FutureTask交给它