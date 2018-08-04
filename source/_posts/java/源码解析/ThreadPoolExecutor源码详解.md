### ThreadPoolExecutor源码详解

#### 一些属性

1. **cl**t

```java
	 private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
```

这个属性前三位存储的是线程池的状态，后面的29位存储的是线程池的工作线程数

线程池的状态：

- **RUNNING(运行状态)**：能接受新提交的任务，并且能处理阻塞队列中的任务
- **SHUTDOWN(关闭状态)**:不能接受新提交的任务，但却可以继续处理阻塞队列中已保存的任务 。 在线程池处于 RUNNING 状态时, 调用 shutdown()方法会使线程池进入到该状态.  
- **STOP**:不能接受新提交的任务, 也不能处理阻塞队列中已保存的任务, 并且会中断正在处理中的任务.  在线程池处于 RUNNING 或 SHUTDOWN 状态时, 调用 shutdownNow() 方法会使线程池进入到该状态.  
- **TIDYING (清理状态):** 所有的任务都已终止了, workerCount (有效线程数) 为0, 线程池进入该状态后会调用 当线程池处于 SHUTDOWN 状态时, 如果此后线程池内没有线程了并且阻塞队列内也没有待执行的任务了 (即: 二者都为空), 线程池就会进入到该状态. 当线程池处于 STOP 状态时, 如果此后线程池内没有线程了, 线程池就会进入到该状态.  
- **TERMINATED :** terminated() 方法执行完后就进入该状态. 

1. **corePoolSize**

   核心线程数，这个属性的意义是，当线程池里面的workerCount<corePoolSize那么提交一个新的任务，都会新建一个线程去处理，然后这些核心线程处理完任务后及时处于空闲状态，也不会对它们消除。

2. **maximumPoolSize** 

   最大线程数，这个属性的意义是线程池最大拥有的线程数，当一个新的任务到达时先是考虑把它添加到阻塞队列里面，如果添加失败，也就是说阻塞队列满了，那么就会创建一个线程去执行任务。但是创建的线程数量加上核心线程的数量不能超过这个值，超过了就应该进行拒绝策略进行拒绝

3. **keepAliveTime**

   非核心线程的空闲保活时间，当创建了一个非核心线程执行任务后，因为没有任务执行一直处理空闲状态，当这个状态的时间超过该属性的值，就会对该线程销毁

4. **workQueue**

   是一个用于保存等待执行任务的阻塞队列，提交任务，对于任务的处理有三种方式，这三种方式其实上面的处理都提到了，这里我总结一下,就是excute()方法的逻辑：

   1. 如果线程池中的线程数小于核心线程数，那么就会创建新的线程执行任务
   2. 如果线程池中的数量大于核心线程数，那么就将任务添加到阻塞队列
   3. 如果队列满了或者其他情况，导致添加失败，就会创建新的线程用于执行任务

   三种处理策略：

   1. 直接切换(使用SynchronizeQueue):当提交一个任务到包含这种 SynchronousQueue 队列的线程池以后, 线程池会去检测是否有可用的空闲线程来执行该任务, 如果没有就直接新建一个线程来执行该任务而不是将该任务先暂存在队列中. “直接切换”的意思就是, 处理方式由”将任务暂时存入队列”直接切换为”新建一个线程来处理该任务”. 这种策略适合用来处理多个有相互依赖关系的任务, 因为该策略可以避免这些任务因一个没有及时处理而导致依赖于该任务的其他任务也不能及时处理而造成的锁定效果. 因为这种策略的目的是要让几乎每一个新提交的任务都能得到立即处理, 所以这种策略通常要求最大线程数 maximumPoolSizes 是无界的(即: Integer.MAX_VALUE). 静态工厂方法 Executors.newCachedThreadPool() 使用了这个队列。  
   2. 使用无界队列：使用无界队列将使得线程池中能够创建的最大线程数就等于核心线程数 corePoolSize, 这样线程池的 maximumPoolSize 的数值起不到任何作用. 当要处理的多个任务之间没有任何相互依赖关系时, 就适合使用这种队列策略来处理这些任务. 静态工厂方法 Executors.newFixedThreadPool() 使用了这个队列。  
   3. 使用有界队列：需要合理的分配最大线程数和队列容量

5. **threadFactory**

   线程构造工厂

6. **handler**

   拒绝策略，拒绝的条件：

   1. 当线程池处于 SHUTDOWN (关闭) 状态时 (不论线程池和阻塞队列是否都已满)  
   2. 当线程池中的所有线程都处于运行状态并且线程池中的阻塞队列已满时 

   具体的拒绝策略

   -  **AbortPolicy**: 	 这是一种直接抛异常的处理方式, 抛出 RejectedExecutionException 异常.  
   - **CallerRunsPolicy**: 将新提交的任务放在 ThreadPoolExecutor.execute()方法所在的那个线程中执行. 
   - **DiscardPolicy**: 直接不执行新提交的任务. 
   - **DiscardOldestPolicy**:  当线程池未关闭时, 会将阻塞队列中处于队首 (head) 的那个任务从队列中移除, 然后再将这个新提交的任务加入到该阻塞队列的队尾 (tail) 等待执行. 

   

   

#### 线程调度

- 首先我们按照逻辑，把其中调度的重要的源码贴出来

  1. execute

     ```java
     public void execute(Runnable command) {
             if (command == null)
                 throw new NullPointerException();
             int c = ctl.get();
             if (workerCountOf(c) < corePoolSize) {
                 if (addWorker(command, true))
                     return;
                 c = ctl.get();
             }
             if (isRunning(c) && workQueue.offer(command)) {
                 int recheck = ctl.get();
                 if (! isRunning(recheck) && remove(command))
                     reject(command);
                 else if (workerCountOf(recheck) == 0)
                     addWorker(null, false);
             }
             else if (!addWorker(command, false))
                 reject(command);
         }
     ```

     这个我在上面已经讲了，就是提交任务后对任务处理的三种情况

  2. addWorker

     ```java
      private boolean addWorker(Runnable firstTask, boolean core) {
             retry:
             for (;;) {
                 int c = ctl.get();
                 int rs = runStateOf(c);
     
                 // Check if queue empty only if necessary.
                 if (rs >= SHUTDOWN &&
                     ! (rs == SHUTDOWN &&
                        firstTask == null &&
                        ! workQueue.isEmpty()))
                     return false;
     
                 for (;;) {
                     int wc = workerCountOf(c);
                     if (wc >= CAPACITY ||
                         wc >= (core ? corePoolSize : maximumPoolSize))
                         return false;
                     if (compareAndIncrementWorkerCount(c))
                         break retry;
                     c = ctl.get();  // Re-read ctl
                     if (runStateOf(c) != rs)
                         continue retry;
                     // else CAS failed due to workerCount change; retry inner loop
                 }
             }
     
             boolean workerStarted = false;
             boolean workerAdded = false;
             Worker w = null;
             try {
                 w = new Worker(firstTask);
                 final Thread t = w.thread;
                 if (t != null) {
                     final ReentrantLock mainLock = this.mainLock;
                     mainLock.lock();
                     try {
                         // Recheck while holding lock.
                         // Back out on ThreadFactory failure or if
                         // shut down before lock acquired.
                         int rs = runStateOf(ctl.get());
     
                         if (rs < SHUTDOWN ||
                             (rs == SHUTDOWN && firstTask == null)) {
                             if (t.isAlive()) // precheck that t is startable
                                 throw new IllegalThreadStateException();
                             workers.add(w);
                             int s = workers.size();
                             if (s > largestPoolSize)
                                 largestPoolSize = s;
                             workerAdded = true;
                         }
                     } finally {
                         mainLock.unlock();
                     }
                     if (workerAdded) {
                         t.start();
                         workerStarted = true;
                     }
                 }
             } finally {
                 if (! workerStarted)
                     addWorkerFailed(w);
             }
             return workerStarted;
         }
     ```

     这个方法是根据线程池的状态和限制判断是否可以添加新线程，如果可以，改变workerCount并且以参数fisrtTask为任务，进行运行；

  3. Worker的run方法内容

     ```java
         final void runWorker(Worker w) {
             Thread wt = Thread.currentThread();
             Runnable task = w.firstTask;
             w.firstTask = null;
             w.unlock(); // allow interrupts
             boolean completedAbruptly = true;
             try {
                 while (task != null || (task = getTask()) != null) {
                     w.lock();
                     // If pool is stopping, ensure thread is interrupted;
                     // if not, ensure thread is not interrupted.  This
                     // requires a recheck in second case to deal with
                     // shutdownNow race while clearing interrupt
                     if ((runStateAtLeast(ctl.get(), STOP) ||
                          (Thread.interrupted() &&
                           runStateAtLeast(ctl.get(), STOP))) &&
                         !wt.isInterrupted())
                         wt.interrupt();
                     try {
                         beforeExecute(wt, task);
                         Throwable thrown = null;
                         try {
                             task.run();
                         } catch (RuntimeException x) {
                             thrown = x; throw x;
                         } catch (Error x) {
                             thrown = x; throw x;
                         } catch (Throwable x) {
                             thrown = x; throw new Error(x);
                         } finally {
                             afterExecute(task, thrown);
                         }
                     } finally {
                         task = null;
                         w.completedTasks++;
                         w.unlock();
                     }
                 }
                 completedAbruptly = false;
             } finally {
                 processWorkerExit(w, completedAbruptly);
             }
         }
     ```

     主worker循环执行这个内容，不断的从队列中获取并执行它们

#### 讲解一下Executors工厂方法构建出来的线程池

1. newFixedThreadPool

   ```java
           return new ThreadPoolExecutor(nThreads, nThreads,
                                         0L, TimeUnit.MILLISECONDS,
                                         new LinkedBlockingQueue<Runnable>());
   ```

根据名称我们知道目的是想构建一个拥有固定线程数的线程池，源码得出以下特点：

1. 核心线程数和最大线程线程数相同
2. 阻塞队列为无界队列

根据以上特点我们知道核心线程数和最大线程数相同那么这个线程池只会创建nThreads的线程，然后根据第第二个特点我们知道这个线程池也不会拒绝任务，所有的任务都会加入无界队列

2. newCachedThreadPool

   ```java
           return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                         60L, TimeUnit.SECONDS,
                                         new SynchronousQueue<Runnable>());
   ```

   根据方法名目的是想创建一个根据需求创建线程的线程池，源码得出以下特点：

   1. 没有核心线程
   2. 最大线程数无穷大
   3. 阻塞队列为SynchronousQueue

   根据这几个特点我们知道这个线程池会为所有新来的任务创建新的线程，而且线程数不受限制

3. newSingleThreadPool 

   ```java
           return new FinalizableDelegatedExecutorService
               (new ThreadPoolExecutor(1, 1,
                                       0L, TimeUnit.MILLISECONDS,
                                       new LinkedBlockingQueue<Runnable>()));
   ```

   根据方法名目的是想创建一个只有一个线程的线程池，源码得出以下特点：

   1. 最大线程数和核心线程数都为1
   2. 阻塞队列为LinkedBlockingQueue

   这个线程池只有用一个线程来完成所有任务，所有任务都添加到阻塞队列里面