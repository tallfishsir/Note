##### Java 多线程

Java实现多线程的方式有：

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
      int count = 0;
      
      @Override
      public Thread newThread(Runnable runnable){
  		count++;
          return new Thread(runnable, "Thread-" + count);
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
  executor,executor(runnable);
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

##### 线程池的常用方法和源码解析

Java 线程池是通过 ThreadPoolExecutor 提供一系列参数来配置线程池。目前，ThreadPoolExecutor共有四个构造函数，下面这个方法是参数最多的，我们一一分析

```java
/**
     * Creates a new {@code ThreadPoolExecutor} with the given initial
     * parameters.
     *
     * @param corePoolSize 核心线程数大小，当提交一个任务时，如果当前县城数目小于它，就会创建一个线程。
     * @param maximumPoolSize 线程池最大容量
     * @param keepAliveTime 线程执行完成后，保持存活的时间
     * @param unit keepAlive 的时间单位
     * @param workQueue 用于保存等待执行的任务的阻塞队列，可以选择：
     *					ArrayBlockingQueue：基于数组的有界阻塞队列，FIFO 原则排序
     *					LinkedBlockingQueue：基于链表的阻塞队列，FIFO 原则排序。
     *					SynchronousQueue：不存储元素的阻塞队列，每个插入操作必须等上一个元素被移除之后。
     * @param threadFactory 设置创建线程的工厂
     * @param handler 线程池、等待队列达到最大容量后的执行策略，默认采用 AbortPolicy，提供了四种方式：
     *					AbortPolicy：直接抛出异常
     *					CallerRunsPolicy：只用调用者所在的线程来执行任务
     *					DiscardOldestPolicy：丢弃队列中最近的一个任务，并执行当前任务
     *					DiscardPolicy：不处理，直接丢弃
     * @throws IllegalArgumentException if one of the following holds:<br>
     *         {@code corePoolSize < 0}<br>
     *         {@code keepAliveTime < 0}<br>
     *         {@code maximumPoolSize <= 0}<br>
     *         {@code maximumPoolSize < corePoolSize}
     * @throws NullPointerException if {@code workQueue}
     *         or {@code threadFactory} or {@code handler} is null
     */
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) 
```

为了调用方便，Executors 提供了常用线程池的静态生成方法：

```java
// 创建一个容量无限大的线程池，线程存活时间 60s
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}

// 创建只含有一个线程的线程池，所有的任务都是串行
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}

// 创建一个固定容量的线程池，需要使用后调用 ExecutorService.shutdown() 关闭线程池
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}

// 创建一个可以延迟执行任务的线程池
public ScheduledThreadPoolExecutor(int corePoolSize) {
    super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
          new DelayedWorkQueue());
}
```

线程池如何实现线程复用，源码解析：



##### 线程同步和线程安全

在多个线程访问共同的资源时，在某一个线程对资源进行写操作的中途，其他线程对这个资源进行了读操作，就会导致数据操作，这就是线程安全问题，为了解决线程安全问题，就有了线程同步的一些方案：

- synchronized 方法
- synchronized 代码块

synchronized 方法是 synchronized 代码块一种简写方式，其中用户同步的对象就是对象本身，但注意静态方法的同步对象只能使用类对象。synchronized 的本质是：1. 保证方法内部或代码块内部资源的互斥访问，同一时间、由同一个 Monitor 监视的代码，最多只能有一个线程访问；2. 保证线程之间对监视资源的数据同步，任何线程在获取到 Monitor 后第一时间会将共享内存中的数据复制到自己的缓存中，在释放 Monitor 的第一时间会先将缓存中的数据复制到共享内存中。

- volatile

保证加了 volatile 关键字的字段的操作具有原子性和同步性，相当于简化版的 synchronized。

- java.util.concurrent.atomic 包

相当于通用版的 volatile。

- Lock/ReentrantReadWriteLock

中断线程的方法可以通过 Thread.interrupt()来设置。

Thread.join Thread.yield

生产消费者

Android 一直循环为什么不崩溃