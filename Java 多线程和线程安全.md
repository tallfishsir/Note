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



Model: Default (GPT-3.5)

> 1 / 1
>
> 详细讲解一下 Java 线程安全的三个要素

1 / 1

Java 线程安全的三个要素是：原子性、可见性、有序性。

1.  原子性：原子性指的是一个操作是不可中断的，要么全部执行成功，要么全部执行失败。Java 提供了 synchronized 关键字和 Lock 接口来保证代码块或方法的原子性。这些机制可以确保同一时刻只有一个线程能够访问共享资源，避免多个线程同时修改同一共享变量而导致的数据竞争和不一致性。

2.  可见性：可见性指的是当一个线程修改了某个共享变量的值，其他线程能够立即看到这个修改。Java 中提供了 volatile 关键字来保证变量的可见性。使用 volatile 关键字修饰的变量的值会被立即刷新到主存中，并且每次读取该变量时都会从主存中读取最新的值，避免了多线程之间的数据不一致问题。

3.  有序性：有序性指的是程序执行的顺序按照代码的先后顺序执行。Java 中提供了 synchronized 和 volatile 关键字来保证程序的有序性。synchronized 关键字会使得代码块中的指令按照代码的先后顺序执行，并且会保证在一个线程释放锁之前，所有对共享变量的修改都会被刷新到主存中。volatile 关键字则可以保证对变量的读写操作按照程序的顺序执行。

综上，Java 线程安全的三个要素是相互关联的，通过它们的结合使用可以保证多线程程序的正确性和可靠性。

> 1 / 1
>
> 在 ReentrantLock.lock 中 ，可以执行 wait 操作吗

1 / 1

在 `ReentrantLock` 的 `lock()` 方法中，是不能执行 `wait()` 操作的。

`ReentrantLock` 使用的是显式锁，它的加锁和释放锁必须手动控制，而且与`synchronized`关键字不同，`ReentrantLock` 并没有自动的监视器锁（monitor lock）的概念，也就是没有像 `wait()` 和 `notify()` 这样的方法。如果在 `ReentrantLock` 中执行 `wait()` 方法，会抛出 `IllegalMonitorStateException` 异常，因为当前线程没有持有监视器锁。

如果需要等待某个条件，可以使用 `Condition` 类中的方法，比如 `await()`、`signal()`、`signalAll()`。它们的作用类似于 `wait()`、`notify()`、`notifyAll()` 方法，但是使用 `Condition` 需要先获取 `ReentrantLock` 对象，通过 `ReentrantLock.newCondition()` 方法创建一个 `Condition` 实例，并且在调用 `await()`、`signal()`、`signalAll()` 方法前必须先持有该 `ReentrantLock` 对象的锁。这样可以确保当前线程在等待某个条件时不会释放锁，而且可以精确地通知等待条件的线程。

> 1 / 1
>
> 「但是使用 Condition 需要先获取 ReentrantLock 对象，通过 ReentrantLock.newCondition() 方法创建一个 Condition 实例，并且在调用 await()、signal()、signalAll() 方法前必须先持有该 ReentrantLock 对象的锁。」举一个例子

1 / 1

假设我们有一个线程安全的队列 `MyQueue`，支持插入元素和取出元素的操作。如果队列为空时，取出操作需要等待插入操作，我们可以使用 `ReentrantLock` 和 `Condition` 来实现等待和唤醒的功能。

示例代码如下：

java

```java
import java.util.LinkedList;
import java.util.Queue;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class MyQueue<T> {
    private Queue<T> queue = new LinkedList<T>();
    private Lock lock = new ReentrantLock();
    private Condition notEmpty = lock.newCondition();

    public void put(T value) {
        lock.lock();
        try {
            queue.offer(value);
            notEmpty.signal(); // 唤醒等待的线程
        } finally {
            lock.unlock();
        }
    }

    public T take() throws InterruptedException {
        lock.lock();
        try {
            while (queue.isEmpty()) { // 队列为空时等待
                notEmpty.await();
            }
            return queue.poll();
        } finally {
            lock.unlock();
        }
    }
}
```

在上面的例子中，`ReentrantLock` 用于保护队列，防止多个线程同时修改队列。在 `put()` 方法中，先获取锁，然后插入元素，最后唤醒等待的线程。在 `take()` 方法中，先获取锁，然后判断队列是否为空，如果为空则调用 `await()` 方法进入等待状态，等待其他线程调用 `signal()` 方法唤醒它。如果队列不为空，则取出队首元素，释放锁，并返回该元素。需要注意的是，在调用 `await()` 和 `signal()` 方法之前，都必须先持有锁，以确保线程安全。



Model: GPT-4

> 1 / 1
>
> 详细讲解一下 Java 线程安全的三个要素

1 / 1

Java 线程安全是指在多线程环境下，程序的行为符合预期，不会出现异常或者错误的结果。实现 Java 线程安全主要涉及三个要素：原子性，可见性和有序性。接下来，我们将详细讨论这三个要素。

1.  原子性（Atomicity） 原子性是指一个操作是不可中断的，要么执行成功，要么完全不执行。在 Java 中，原子性操作通常是通过以下几种方式来保证的：

a. 使用原子类（如 AtomicInteger、AtomicLong 等），这些类的方法是原子性的，可以在多线程环境下安全地进行操作。 b. 使用 synchronized 关键字，它可以锁定一个对象，确保同一时刻只有一个线程可以访问这个对象的代码块。这样就能保证代码块中的操作具有原子性。 c. 使用 Lock 接口及其实现类（如 ReentrantLock），提供了与 synchronized 类似的功能，但是更灵活。

2.  可见性（Visibility） 可见性是指一个线程对共享变量的修改，能够及时地被其他线程感知到。在 Java 中，可以通过以下方式来保证可见性：

a. 使用 volatile 关键字修饰共享变量。volatile 关键字可以确保一个线程对变量的修改对其他线程立刻可见，从而保证可见性。 b. 使用 synchronized 关键字，它除了保证原子性外，还具有内存屏障的作用，确保锁定的共享变量对其他线程的修改在解锁时立即可见。 c. 使用原子类（如 AtomicInteger、AtomicLong 等），原子类内部实现了可见性的保证。

3.  有序性（Ordering） 有序性是指程序执行的顺序按照代码的先后顺序进行。在多线程环境下，由于编译器优化和处理器优化，指令可能会发生重排序，导致代码执行顺序与预期不符。Java 中可以通过以下方式来保证有序性：

a. 使用 volatile 关键字。volatile 关键字不仅保证可见性，还阻止了编译器和处理器对其修饰的变量进行指令重排序。 b. 使用 synchronized 关键字。synchronized 关键字同样具有防止指令重排序的作用，确保代码块内的操作按照顺序执行。 c. 使用 java.util.concurrent 包下的并发工具类（如 CountDownLatch、CyclicBarrier 等），这些工具类可以协调多个线程之间的执行顺序。

总之，实现 Java 线程安全需要关注原子性、可见性和有序性这三个要素。通过使用原子类、synchronized 关键字、Lock 接

> 1 / 1
>
> 继续

1 / 1

口、volatile 关键字以及 java.util.concurrent 包下的工具类，可以在多线程环境下保证代码的正确执行。

在实际开发中，要根据具体场景选择合适的方法来保证线程安全。以下是一些建议：

1.  尽量使用不可变类。不可变类的实例一旦创建，其状态就不能改变。由于不可变类本身就是线程安全的，因此在多线程环境下可以大大简化编程。

2.  使用局部变量。局部变量只在当前线程的栈上分配空间，因此它们是线程安全的。尽量避免使用全局变量或共享变量，以减少线程间同步的需求。

3.  使用并发容器。Java 提供了很多线程安全的集合类，如 ConcurrentHashMap、CopyOnWriteArrayList 等。它们在保证线程安全的同时，也提供了较高的性能。

4.  使用线程封闭。线程封闭是指将数据封闭在某个线程中，使其他线程无法访问。可以通过 ThreadLocal 类实现线程封闭，保证每个线程都有自己的数据副本。

5.  使用锁分离。锁分离是指将一个大的锁拆分成几个小锁，以减少锁竞争，提高并发性能。例如，可以使用 ReadWriteLock 代替 synchronized，使读操作和写操作分别使用不同的锁。

6.  使用线程池。线程池可以有效地控制线程的数量，减少线程间的竞争。此外，线程池还可以复用线程，降低线程创建和销毁的开销。

7.  分析并发性能。在实际开发中，要通过性能测试和分析，找出代码的性能瓶颈，有针对性地进行优化。注意，过度优化往往会带来更多的问题，应在保证代码可读性和可维护性的基础上进行优化。

综上所述，要实现 Java 线程安全，需要关注原子性、可见性和有序性，并结合实际场景，采用合适的方法保证多线程环境下代码的正确执行。同时，要注意代码的性能和可维护性。

> 1 / 1
>
> 在 ReentrantLock.lock 中 ，可以执行 wait 操作吗

1 / 1

在 ReentrantLock 中，我们不能直接使用 `wait()` 操作，因为 `wait()` 方法是 `Object` 类的一个方法，用于配合 `synchronized` 关键字进行线程间通信。在使用 `synchronized` 锁时，可以使用 `wait()`、`notify()` 和 `notifyAll()` 方法进行线程间的等待与通知。

然而，ReentrantLock 提供了一个类似的机制：Condition。Condition 可以与 ReentrantLock 配合使用，实现类似的等待与通知功能。在使用 ReentrantLock 时，你可以使用以下步骤来实现等待与通知功能：

1.  创建 ReentrantLock 实例。

java

```java
ReentrantLock lock = new ReentrantLock();
```

2.  使用 ReentrantLock 实例创建一个 Condition 对象。

java

```java
Condition condition = lock.newCondition();
```

3.  获取锁。

java

```java
lock.lock();
```

4.  使用 Condition 对象的 `await()` 方法进行等待。这将释放锁并使当前线程等待，直到其他线程调用 `signal()` 或 `signalAll()` 方法。

java

```java
try {
    condition.await();
} catch (InterruptedException e) {
    // Handle the interruption
} finally {
    lock.unlock();
}
```

5.  在另一个线程中，使用 Condition 对象的 `signal()` 或 `signalAll()` 方法进行通知。`signal()` 方法唤醒一个等待的线程，而 `signalAll()` 方法唤醒所有等待的线程。

java

```java
lock.lock();
try {
    condition.signal(); // 或者 condition.signalAll();
} finally {
    lock.unlock();
}
```

总之，虽然在 ReentrantLock 中不能直接使用 `wait()` 方法，但可以通过创建与 ReentrantLock 关联的 Condition 对象来实现类似的等待与通知功能。