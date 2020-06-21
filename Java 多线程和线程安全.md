##### Q1：java 开启多线程的方法有哪些？

- 重写 Thread 的 run() 方法

  ```java
  Thread thread = new Thread(){
      @Override
      public void run(){
  		// do something
      }
  }
  thread.start();
  ```

- 创建 Runnable 对象

  ```java
  Runnable runnable = new Runnable(){
      @Override
      public void run(){
  		// do something
      }
  }
  Thread thread = new Thread(runnable);
  thread.start();
  ```

- 重写 ThreadFactory 的 newThread(Runnable runnable) 方法

  ```java
  ThreadFactory factory = new ThreadFactory(){
      @Override
      public Thread newThread(Runnable runnable){
          return new Thread(runnable, "New Thread");
      }
  }
  Runnable runnable = new Runnable(){
      @Override
      public void run(){
  		// do something
      }
  }
  Thread thread = factory.newThread(runnable);
  thread.start();
  ```

- Executor 和线程池

  ```java
  Executor executor = Executors.newCacheThreadPool();
  Runnable runnable = new Runnable(){
      @Override
      public void run(){
  		// do something
      }
  }
  executor.executor(runnable);
  ```

- Callable 和 Future

  ```java
  Callable<String> callable = new Callable<String>{
      @Override
      public String call(){
  		// do something
          return "done";
      }
  }
  Executor executor = Executors.newCacheThreadPool();
  Future<String> future = executor.submit(callable);
  String result = future.get();
  ```

##### Q2：ThreadPoolExecutor 的参数含义及 Executors 的静态方法？

线程池是通过 ThreadPoolExecutor 提供的一系列参数来配置创建的。关键参数有：

- corePoolSize —— 核心线程数

  - 当有一个新任务时，如果当前线程数小于核心线程数时，即使有核心线程空闲，也会创建新的线程。

  - allowCoreThreadTimeout 默认 false，核心线程会一直存活；设置 true，核心线程空闲超时会回收。

- workQueue —— 阻塞队列

  - 如果当前的核心线程数已经达到最大，无论是否核心线程中是否有空闲线程，新的任务会被放到队列中排队等待被执行。

  - 一般可以选择：ArrayBlockingQueue（基于数组的有界阻塞队列）、LinkedBlockingQueue（基于链表的阻塞队列，不存在最大值限制）、SynchronousQueue（不存储元素的阻塞队列，每个插入操作必须等上一个元素被移除）

- maximumPoolSize —— 最大线程数

  - 当前线程数＞corePoolSize，且阻塞队列已满时，线程池会创建新的线程来处理新的任务

  - 当前线程数＝maximumPoolSize，且阻塞队列已满时，线程池会拒绝处理新任务，并根据 RejectedExecutionHandler 来处理

- keepAliveTime —— 线程超时时间

  - 当前线程数＞corePoolSize，当非核心线程空闲时间达到 keepAliveTime 时会退出线程

  - 如果 allowCoreThreadTimeout＝true，核心线程空闲时间达到 keepAliveTime 时会退出线程

- TimeUnit —— 时间单位

  - TimeUnit 是一个枚举类型，常用的时间单位有：MILLISECOND SECONDS MINUTES HOURS DAYS

- rejectedExecutionHandler —— 任务拒绝处理器

  - 有两种情况会启动 rejectedExecutionHandler
    1. 当前线程数＝maxPoolSize，且阻塞队列已满时会启动
    2. 线程池调用 shutdown() 后，会等待阻塞队列任务执行完再关闭，此时有新的任务会启动
  - 共有四种策略，默认采用了 AbortPolicy
    1. AbortPolicy：丢弃任务，并抛出运行时异常
    2. CallerRunsPolicy：只用调用者所在的线程来处理任务
    3. DiscardOldestPolicy：丢弃阻塞队列中最近的任务，然后执行当前任务
    4. DiscardPolicy：直接丢弃任务，不处理

总结来说，新任务到达后，线程池执行任务的行为顺序是：

1. 当前线程数＜corePoolSize，创建线程
2. 当前线程数＝corePoolSize，放入阻塞队列中排队
3. 当前线程数＝corePoolSize，阻塞队列已满，创建新的线程
4. corePoolSize＜当前线程数＜maximumPoolSize，阻塞队列未满，放入队列，阻塞队列已满，创建新的线程
5. 当前线程数＝maximumPoolSize，阻塞队列已满，执行 rejectedExecutionHandler

为了方便开发者创建线程池，Executors 提供了四个静态方法用于创建不同特点的线程池：

```java
/**
 * 创建一个容量无限大的线程池，线程存活时间 60s
 * 需要执行很多短时间的任务时，newCachedThreadPool 的线程复用率比较高，会显著的提高性能
 */
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}

/**
 * 创建只含有一个线程的线程池，所有的任务都是串行
 */
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}

/**
 * 创建一个固定容量的线程池，需要使用后调用 ExecutorService.shutdown() 关闭线程池
 */
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}

/**
 * 创建一个可以延迟执行任务的线程池
 */
public ScheduledThreadPoolExecutor(int corePoolSize) {
    super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
          new DelayedWorkQueue());
}
```

##### Q3：ThreadPoolExecutor 源码解析？

ThreadPoolExecutor 中常用的成员变量和常量

```java
// 低29位表示线程池中线程数，通过高3位表示线程池的运行状态
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
private static final int COUNT_BITS = Integer.SIZE - 3;
// 线程池的最大容量（00011111111111111111111111111111）
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

// 此状态可接收新任务或处理阻塞队列中的任务（11100000000000000000000000000000）
private static final int RUNNING    = -1 << COUNT_BITS;
// 此状态不会接收新任务，但会处理阻塞队列中的任务（00000000000000000000000000000000）
private static final int SHUTDOWN   =  0 << COUNT_BITS;
// 此状态不会接收新任务，不会处理阻塞队列中的任务，而且会中断正在运行的任务 （00100000000000000000000000000000）
private static final int STOP       =  1 << COUNT_BITS;
// 此状态所有线程已停止，准备执行terminated()方法（01000000000000000000000000000000）
private static final int TIDYING    =  2 << COUNT_BITS;
// 此状态表示已执行完terminated()方法（01100000000000000000000000000000）
private static final int TERMINATED =  3 << COUNT_BITS;

/**
 * 通过ctl的高3位值，获取线程池状态
 */
private static int runStateOf(int c)     { return c & ~CAPACITY; }
/**
 * 通过ctl的低29位值，获取线程池中的线程数，最大数量是 2^29-1
 */
private static int workerCountOf(int c)  { return c & CAPACITY; }
```

先了解下 AtomicInteger 原子操作类是如何保证在多线程的环境下保证线程安全。它是位于 java.util.concurrent.atomic 包下的包装类，用于 Integer 类型的原子性操作。常见的自增自减方法利用了 CAS 机制（Compare And Swap）。

线程池的源码分析，从调用 execute() 方法处理任务开始：

```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    int c = ctl.get();
    // 如果当前线程数＜corePoolSize，创建新的线程执行任务
    if (workerCountOf(c) < corePoolSize) {
        // addWork() 返回true表示创建成功，返回false表示创建失败
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    // 线程池状态是RUNNING 当前线程数＞corePoolSize 就将任务加入阻塞队列中
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        // 再次检查线程池状态，如果线程池不是RUNNING状态，就从阻塞队列中删除任务，然后执行rejectedExecutionHandler处理任务
        if (!isRunning(recheck) && remove(command))
            reject(command);
        // 再次检查线程池中的线程数，防止正好线程正在关闭而阻塞队列中还有任务的特殊情况，创建一个新的线程
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    // 到流程到此，有可能是线程池不在RUNNING状态或者阻塞队列已满，就添加非核心线程
    // 所以就创建新的线程去执行任务，如果创建线程失败，就执行rejectedExecutionHandler处理任务
    else if (!addWorker(command, false))
        reject(command);
}
```

由 execute 的流程中可以看出，addWorker 方法完成了线程池主要的创建线程和执行任务的功能。

```java
/**
 * 返回true表示创建线程成功，返回false表示没有创建新的线程
 */
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
        int c = ctl.get();
        // 获取当前线程状态
        int rs = runStateOf(c);
        // 如下几种情况，都不会创建新的线程：
        // 1.线程池状态是 STOP TIDYING TERMINATED 此时不创建新的线程处理任务
        // 2.线程池状态是 SHUTDOWN & 有新的任务，此时不创建新的线程处理任务，但是仍会执行队列中的任务
        // 3.线程池状态是 SHUTDOWN & 阻塞队列为空，此时不创建新的线程处理任务。多这个判断是为了防止了SHUTDOWN 状态下没有活动线程了，但是队列里还有任务没执行这种特殊情况，就开启一个新的线程执行任务
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
            return false;
        for (;;) {
            int wc = workerCountOf(c);
            // 如果当前线程数大于了最大线程数 或者 当前线程数大于了相应的 corePoolSize/maximumPoolSize，不会创建新的线程处理任务
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            // CAS 来安全增加当前线程数并检查线程池状态
            if (compareAndIncrementWorkerCount(c))
                break retry;
            c = ctl.get();  // Re-read ctl
            if (runStateOf(c) != rs)
                continue retry;
        }
    }
    
    // 这里已经成功通过CAS操作将当前线程数加一，下面创建线程 
    boolean workerStarted = false;
    boolean workerAdded = false;
    // Worker实现了Runnable接口，有两个成员变量：Thread thread；Runnable firstTask；
    Worker w = null;
    try {
        // 在构造函数中 thread = getThreadFactory().newThread(this); 创建一个新的线程
        // firstTask 则是线程池中新添加的任务
        w = new Worker(firstTask);
        final Thread t = w.thread;
        if (t != null) {
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            // 在读写锁中把创建的worker对象添加到workers（HashSet）中。
            try {
                int rs = runStateOf(ctl.get());
                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    if (t.isAlive())
                        throw new IllegalThreadStateException();
                    // 将新启动的线程添加到线程池中
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
                // 首先执行 Worker 的 run()，其中调用了 Worker 的 runWoker()
                // 在runWorker()中，会先执行 firstTask，后不停的从队列中取任务执行
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        // 最终发，则从 wokers 中移除 w 并递减 wokerCount
        if (! workerStarted)
            // 递减wokerCount会触发tryTerminate方法
            addWorkerFailed(w);
    }
    return workerStarted;
}
```

runWorker()中，会先执行 firstTask，然后不停的从队列中取任务执行。在从队列中获取任务的方法 getTask() 中，如果队列为空则等待 keepAliveTime 然后返回null，然后在 runWorker 中执行processWorkerExit 关闭线程。

```java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
        // 第一次 task 不为空，执行 run 方法，
        // 由于 while 循环，会通过 getTask() 方法获取阻塞队列中等待的任务，如果没有任务，getTask会被阻塞挂起
        while (task != null || (task = getTask()) != null）{
            w.lock();
            if ((runStateAtLeast(ctl.get(), STOP) || (Thread.interrupted() && runStateAtLeast(ctl.get(), STOP))) 
                && !wt.isInterrupted())
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
               
private Runnable getTask() {
    boolean timedOut = false;
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            // 返回 null 让 runWorker 退出 while 循环也就是当前线程结束了，所以必须要 decrement 递减
            decrementWorkerCount();
            return null;
        }
        int wc = workerCountOf(c);
        // 如果当前线程数＞corePoolSize 或者 允许核心线程空闲超时
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
        if ((wc > maximumPoolSize || (timed && timedOut)) && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }
        try {
            // 1.以指定的超时时间从队列中取任务
            // 2.core thread没有超时
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            if (r != null)
                return r;
            timedOut = true;
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}
```

##### Q4：多线程编程中线程安全三个核心概念？

线程安全有三个核心概念：

- 原子性：一个操作（可能含有多个子操作）要么全部执行，要么全部不执行
- 可见性：多个线程并发访问共享变量时，一个线程对共享变量的修改，其他线程可以立即看到。
- 顺序性：程序执行的顺序时按照代码的先后顺序执行

###### 如何保证原子性？

常用的保证操作原子性的工具是锁和同步方法（或者同步代码块），通过锁来实现资源的排他性，从而目标代码同一时间只会被一个线程执行，是一种以牺牲性能为代价的方法。

使用锁，可以保证同一时间只有一个线程能拿到锁，也就保证同一时间只有一个线程能执行申请锁和释放锁之间的代码。为了提高读写操作的效率，Java 提供了读写锁（ReadWriteLock），ReentrantReadWriteLock 也只实现了ReadWriteLock接口而未实现Lock接口。它内部有两个静态内部类：ReadLock和WriteLock，它们实现了Lock接口。获得读锁后，其它线程可获得读锁而不能获取写锁，获得写锁后，其它线程既不能获得读锁也不能获得写锁。

与锁类似的是 synchronized 方法，使用非静态同步方法，锁住的是当前实例；使用静态同步方法，锁住的是该类的 Class 对象；使用静态代码块，锁住的是 synchronized 关键字后的对象。synchronized 的本质是：

1. 保证方法内部或代码块内部资源的互斥访问，同一时间、由同一个 Monitor 监视的代码，最多只能有一个线程访问（保证原子性）；
2.  保证线程之间对监视资源的数据同步，任何线程在获取到 Monitor 后第一时间会将共享内存中的数据复制到自己的缓存中，在释放 Monitor 的第一时间会先将缓存中的数据复制到共享内存中。（保证可见性）

另外，Java 种还提供了基础类型变量的原子操作类来保证原子性，其本质是利用了 CPU 级别的 CAS 执行，开销比操作系统参与的锁的开销小。

###### 如何保证可见性？

happens-before 用于描述两个操作的内存可见性：如果一个操作 A happens-before 另一个操作 B，那么操作 A 的执行结果将对操作 B 可见。反过来理解就是：如果操作 A 的结果需要对另一个操作 B 可见，那么操作 A 必须 happens-before 操作 B。

JMM Java 内存模型中定义了以下几种情况是自动符合happens-before 原则的：

- 程序次序规则

  单线程内部，如果一段代码的字节码顺序靠前的执行结果一定是对后续字节码可见。

- 锁定规则

  无论是单线程还是多线程环境，一个锁如果处于被锁定的状态，那么必须先执行 unlock 操作才能进行 lock 操作。

- 变量规则

  Java 提供了 volatile 关键字来保证可见性，当使用 volatile 修饰某个变量时，会保证对该变量的修改会立即被更新到主内存中，并且将其他缓存中变量的缓存设置为无效，因此其它线程需要读取该值时必须从主内存中读取，从而得到最新的值。

- 线程启动规则

  Thread 对象的 start 方法先发生于此线程的每一个动作。

- 线程中断规则

  对线程 interrupt 方法的调用会先行发生于被中断线程的代码检测，知道中断事件的发生。

- 线程终结规则

  线程中所有的操作都先行发生于线程的终止检测之前，可以通过 Thread.join() 方法结束。