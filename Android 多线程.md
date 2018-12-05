Android 提供了四种常用的操作多线程的方式

##### ThreadPoolExecutor

ThreadPoolExecutor 类常用的成员常量和成员变量：

```java
// 利用低29位表示线程池中线程数，通过高3位表示线程池的运行状态
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
private static final int COUNT_BITS = Integer.SIZE - 3;
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

// 高3位为111，该状态的线程池会接收新任务，并处理阻塞队列中的任务
private static final int RUNNING    = -1 << COUNT_BITS;
// 高3位为000，该状态的线程池不会接收新任务，但会处理阻塞队列中的任务
private static final int SHUTDOWN   =  0 << COUNT_BITS;
// 高3位为001，该状态的线程不会接收新任务，也不会处理阻塞队列中的任务，而且会中断正在运行的任务
private static final int STOP       =  1 << COUNT_BITS;
// 高3位为010，所有线程已停止，准备执行terminated()方法
private static final int TIDYING    =  2 << COUNT_BITS;
// 高3位为011，表示已执行完terminated()方法
private static final int TERMINATED =  3 << COUNT_BITS;

// 获取 ctl 高3位，线程池的运行状态
private static int runStateOf(int c)     { return c & ~CAPACITY; }
// 获取 ctl 低29位，获取线程池中的线程数，所以线程池最大数量就是2^29-1
private static int workerCountOf(int c)  { return c & CAPACITY; }
```

创建线程池后，需要调用 execute 方法来执行任务

```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();

    int c = ctl.get();
    // 如果当前线程数小于核心线程数，开启核心线程执行任务
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    // 当前线程数不小于核心线程数且把提交的任务成功放入阻塞队列中
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        // 再次检查线程池状态，如果线程池没有RUNNING，且成功从阻塞队列中删除任务，则执行reject方法处理任务
        if (! isRunning(recheck) && remove(command))
            // 移除成功，拒绝该非运行的任务
            reject(command);
        else if (workerCountOf(recheck) == 0)
            // 防止了SHUTDOWN状态下没有活动线程了，但是队列里还有任务没执行这种特殊情况。
            // 添加一个null任务是因为SHUTDOWN状态下，线程池不再接受新任务
            addWorker(null, false);
    }
    // 当前线程数不小于核心线程数且阻塞队列已满就创建新的线程执行任务
    else if (!addWorker(command, false))
        reject(command);
}
```

可以看出来， addWorker 方法主要负责了创建新线程和执行任务的功能。

```java
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
        int c = ctl.get();
        // 获取当前线程状态
        int rs = runStateOf(c);
        // 逻辑判断成立可以分为以下几种情况均不接受新任务
        // 1、rs > shutdown:--不接受新任务
        // 2、rs >= shutdown && firstTask != null:--不接受新任务
        // 3、rs >= shutdown && workQueue.isEmppty:--不接受新任务
        // 逻辑判断不成立
        // 1、rs==shutdown&&firstTask != null:此时不接受新任务，但是仍会执行队列中的任务
        // 2、rs==shotdown&&firstTask == null:会执行addWork(null,false)
        // 防止了SHUTDOWN状态下没有活动线程了，但是队列里还有任务没执行这种特殊情况。
        // 添加一个null任务是因为SHUTDOWN状态下，线程池不再接受新任务
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
            // 线程安全增加工作线程数
            if (compareAndIncrementWorkerCount(c))
                break retry;
            c = ctl.get();  // Re-read ctl
            if (runStateOf(c) != rs)
                continue retry;
            // else CAS failed due to workerCount change; retry inner l
        }
    }
    
    // 这里已经成功执行了CAS操作将线程池数量+1，下面创建线程 
    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        // Worker.thread 就是 Worker本身，firstTask 赋值给 Worker.firstTask 字段
        w = new Worker(firstTask);
        final Thread t = w.thread;
        if (t != null) {
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                // RUNNING状态 || SHUTDONW状态下清理队列中剩余的任务
                int rs = runStateOf(ctl.get());
                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    if (t.isAlive()) // precheck that t is startable
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
            // 启动新添加的线程，这个线程首先执行firstTask，然后不停的从队列中取任务执行
            if (workerAdded) {
                // 执行ThreadPoolExecutor的runWoker方法
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        // 线程启动失败，则从 wokers 中移除 w 并递减 wokerCount
        if (! workerStarted)
            // 递减wokerCount会触发tryTerminate方法
            addWorkerFailed(w);
    }
    return workerStarted;
}
```

addWorker 之后是 runWorker，第一次启动会执行初始化传进来的任务 firstTask；然后会从 workQueue 中取任务执行，如果队列为空则等待 keepAliveTime 这么长时间。

```java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
        // 第一次 task 不为空，执行 run 方法，
        // 由于 while 循环，会通过 getTask() 方法获取阻塞队列中等待的任务，如果没有任务，getTask 会被阻塞挂起
        while (task != null || (task = getTask()) != null
            w.lock();
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
               
private Runnable getTask() {
    boolean timedOut = false; // Did the last poll() time out?
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            // 返回 null 让 runWorker 退出 while 循环也就是当前线程结束了，所以必须要 decrement 递减
            decrementWorkerCount();
            return null;
        }
        int wc = workerCountOf(c);
        // timed 当前线程数大于核心线程数
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
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

##### Handler + Thread

通过 Handler ，可以将任务转移到 Handler 所在的线程执行。底层的实现是利用 Looper 和 MessageQueue 来完成轮询执行的。MessageQueue 存放了需要执行的任务，Looper 的 loop 方法开启循环查询消息队列，如果队列为空，就会阻塞等待。为了保证每个线程都取到属于自己线程的 Looper 对象，Looper 内有一个特殊的对象ThreadLocal\<Looper>。

ThreadLocal 是如何实现同一个对象，在不同的线程中能够获取不同的值呢，先分析下 ThreadLocal get/set 方法

```java
// 首先明确一点，每个 Thread 对象都有一个 ThreadLocal.ThreadLocalMap 对象
// ThreadLocal.ThreadLocalMap 内部使用数组 table 来存放当前线程的所有 ThreadLocal 数据
// ThreadLocalMap.Entry 是一种弱引用，存放在 table 数组的位置是根据 key.threadLocalHashCode & (table.length - 1) 取模操作来得到的
// 总结来说，每个 Thread 内有一个保存 ThreadLocal 对象的数组，数据在数组的位置是根据 hashcode 取模操作得到。
class Thread implements Runnable {
    ThreadLocal.ThreadLocalMap threadLocals = null;
}

static class ThreadLocalMap {
    
    static class Entry extends WeakReference<ThreadLocal> {
        Object value;
        Entry(ThreadLocal k, Object v) {
            super(k);
            value = v;
        }
    }
    
    private Entry getEntry(ThreadLocal key) {
        int i = key.threadLocalHashCode & (table.length - 1);
        Entry e = table[i];
        if (e != null && e.get() == key)
            return e;
        else
            return getEntryAfterMiss(key, i, e);
    }
    
    private void set(ThreadLocal key, Object value) {
        Entry[] tab = table;
        int len = tab.length;
        int i = key.threadLocalHashCode & (len-1);
        for (Entry e = tab[i];
             e != null;
             e = tab[i = nextIndex(i, len)]) {
            ThreadLocal k = e.get();
            if (k == key) {
                e.value = value;
                return;
            }
            if (k == null) {
                replaceStaleEntry(key, value, i);
                return;
            }
        }
        tab[i] = new Entry(key, value);
        int sz = ++size;
        if (!cleanSomeSlots(i, sz) && sz >= threshold)
            rehash();
    }
}
```

现在了解了同一个对象在不同线程是如何存放不同的数据的，再分析下 Looper 源码

```java
public final class Looper {
    // 存放 Looper 的对象
    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
    // 消息队列
    final MessageQueue mQueue;
    // 当前线程
    final Thread mThread;
    
    private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
    }
    
    public static void prepare() {
        prepare(true);
    }

    // 创建当前线程的 Looper 对象，并放入 Thread.ThreadLocal 中
    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }
    
    // 开启循环取 MessageQueue 的消息
    public static void loop() {
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();

        for (;;) {
            // queue.next()有可能会造成阻塞
            Message msg = queue.next();
            if (msg == null) {
                // 如果 msg 为空，说明是 MessageQueue 退出了
                return;
            }

            final Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " + msg.callback + ": " + msg.what);
            }
            final long traceTag = me.mTraceTag;
            if (traceTag != 0) {
                Trace.traceBegin(traceTag, msg.target.getTraceName(msg));
            }
            try {
                // msg.target 在稍后的 Message 源码中可以知道就是 Handler 对象，来处理任务
                msg.target.dispatchMessage(msg);
            } finally {
                if (traceTag != 0) {
                    Trace.traceEnd(traceTag);
                }
            }

            if (logging != null) {
                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
            }
            final long newIdent = Binder.clearCallingIdentity();
            if (ident != newIdent) {
                Log.wtf(TAG, "Thread identity changed from 0x"
                        + Long.toHexString(ident) + " to 0x"
                        + Long.toHexString(newIdent) + " while dispatching to "
                        + msg.target.getClass().getName() + " "
                        + msg.callback + " what=" + msg.what);
            }

            msg.recycleUnchecked();
        }
    }
}
```

上面说到 MessageQueue.next 方法会造成阻塞，下面分析下 MessageQueue 源码

```java
//  MessageQueue 添加元素，MessageQueue 的底层数据结构是单链表
boolean enqueueMessage(Message msg, long when) {
    if (msg.target == null) {
        throw new IllegalArgumentException("Message must have a target.");
    }
    if (msg.isInUse()) {
        throw new IllegalStateException(msg + " This message is already in use.");
    }
    synchronized (this) {
        // 判断当前状态，如果正在结束，就不接受任务
        if (mQuitting) {
            IllegalStateException e = new IllegalStateException(
                    msg.target + " sending message to a Handler on a dead thread");
            Log.w(TAG, e.getMessage(), e);
            msg.recycle();
            return false;
        }
        msg.markInUse();
        msg.when = when;
        // mMessages 是单链表的头结点
        Message p = mMessages;
        boolean needWake;
        if (p == null || when == 0 || when < p.when) {
            // 如果没有头结点，或者执行任务延迟时间 when 是0，或者延迟时间小于头结点的延迟时间
            // 就将当前结点设置为头结点
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked;
        } else {
            // 插入到 when 适合的地方
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            Message prev;
            for (;;) {
                prev = p;
                p = p.next;
                if (p == null || when < p.when) {
                    break;
                }
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
            msg.next = p; // invariant: p == prev.next
            prev.next = msg;
        }
        if (needWake) {
            nativeWake(mPtr);
        }
    }
    return true;
}

// 获取单链表的下一个结点，有可能造成阻塞
Message next() {
    final long ptr = mPtr;
    if (ptr == 0) {
        return null;
    }
    int pendingIdleHandlerCount = -1; // -1 only during first iteration
    int nextPollTimeoutMillis = 0;
    // 开启循环取消息
    for (;;) {
        if (nextPollTimeoutMillis != 0) {
            Binder.flushPendingCommands();
        }
        nativePollOnce(ptr, nextPollTimeoutMillis);
        synchronized (this) {
            // 获取当前事件，用于对比延迟任务的when
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
            if (msg != null && msg.target == null) {
                // Stalled by a barrier.  Find the next asynchronous message in the queue.
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            if (msg != null) {
                if (now < msg.when) {
                    // 延迟实现
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    // 获取消息
                    mBlocked = false;
                    if (prevMsg != null) {
                        prevMsg.next = msg.next;
                    } else {
                        mMessages = msg.next;
                    }
                    msg.next = null;
                    if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                    msg.markInUse();
                    return msg;
                }
            } else {
                nextPollTimeoutMillis = -1;
            }
            ...
        }
        ...
    }
}
```

现在消息生成和循环获取传递给 Handler 的流程分析完了，接下来看下 Handler 是如何 send Message 并处理的

```java
// Handler 的各种 post\sendxxx 方法，最后都会调用这个方法把 Message 放入 MessageQueue 中，并设置msg.target
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    msg.target = this;
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}

// 在 Looper 的 loop 方法中调用了 msg.target.dispatchMessage 来处理消息
public void dispatchMessage(Message msg) {
    if (msg.callback != null) {
        handleCallback(msg);
    } else {
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        handleMessage(msg);
    }
}
```

至此，Handler + Looper + MessageQueue 的多线程机制分析完成。

Activity.runOnUiThread 和 View.post 等方法，底层都是通过获取主线程的 Handler 来实现的。

##### AsyncTask

AsyncTask 使用时有两个注意点：

1. 一个对象只能执行一次 executor 方法;
2. 同一个 AsyncTask 类的对象，会共享一个任务队列

```java
// AsyncTask 构造函数构建 Callable 和 Future 两个对象
public AsyncTask() {
	// WorkRunnable 是 Callable 的子类
    mWorker = new WorkerRunnable<Params, Result>() {
        public Result call() throws Exception {
            mTaskInvoked.set(true);
            Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
            // 后台执行的回调
            Result result = doInBackground(mParams);
            Binder.flushPendingCommands();
			// postResult 会通过 Handler 执行 finish 方法
            return postResult(result);
        }
    };
    mFuture = new FutureTask<Result>(mWorker) {
        @Override
        protected void done() {
            try {
				// 如果 cancel 或者异常就会使用这个方法通知 Handler
                postResultIfNotInvoked(get());
            } catch (InterruptedException e) {
                android.util.Log.w(LOG_TAG, e);
            } catch (ExecutionException e) {
                throw new RuntimeException("An error occurred while executing doInBackground()", e.getCause());
            } catch (CancellationException e) {
                postResultIfNotInvoked(null);
            }
        }
    };
}

private static class InternalHandler extends Handler {
    public InternalHandler() {
        super(Looper.getMainLooper());
    }
    @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
    @Override
    public void handleMessage(Message msg) {
        AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
        switch (msg.what) {
            case MESSAGE_POST_RESULT:
                result.mTask.finish(result.mData[0]);
                break;
            case MESSAGE_POST_PROGRESS:
                result.mTask.onProgressUpdate(result.mData);
                break;
        }
    }
}

private void finish(Result result) {
    if (isCancelled()) {
		// cancel 回调
        onCancelled(result);
    } else {
		// 执行完成回调
        onPostExecute(result);
    }
    mStatus = Status.FINISHED;
}

// 开始启动任务
public final AsyncTask<Params, Progress, Result> execute(Params... params) {
    return executeOnExecutor(sDefaultExecutor, params);
}

public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
        Params... params) {
    if (mStatus != Status.PENDING) {
        switch (mStatus) {
            case RUNNING:
                throw new IllegalStateException("Cannot execute task:"
                        + " the task is already running.");
            case FINISHED:
                throw new IllegalStateException("Cannot execute task:"
                        + " the task has already been executed "
                        + "(a task can be executed only once)");
        }
    }
    mStatus = Status.RUNNING;
    // 执行任务之前回调
    onPreExecute();
    mWorker.mParams = params;
    exec.execute(mFuture);
    return this;
}

private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;
public static final Executor SERIAL_EXECUTOR = new SerialExecutor();
private static class SerialExecutor implements Executor {
    final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
    Runnable mActive;
    public synchronized void execute(final Runnable r) {
        // 内部 Executor 添加到队列中
        mTasks.offer(new Runnable() {
            public void run() {
                try {
                    // r.run 执行 mFuture.run 执行 mWorker.call
                    r.run();
                } finally {
                    scheduleNext();
                }
            }
        });
        if (mActive == null) {
            scheduleNext();
        }
    }
    // THREAD_POOL_EXECUTOR 是线程池 开启线程执行任务
    protected synchronized void scheduleNext() {
        if ((mActive = mTasks.poll()) != null) {
            THREAD_POOL_EXECUTOR.execute(mActive);
        }
    }
}
```

##### IntentService

首先明确在一个 Thread 中是没有 Handler 和 Looper 的，而在一个 Thread 中如果要创建 Handler，必须已经存在 Looper 实例对象。所以，一般情况在 Thread 创建 Handler 之前都会调用 Looper.prepare() 生成一个 Looper 对象。

```java
public abstract class IntentService extends Service {
    @Override
    public void onCreate() {
        super.onCreate();
        // HandlerThread 一个已经创建 Looper 的线程对象
        HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
        // 调用 Looper.prepare() 和 loop() 开启轮询
        thread.start();

        mServiceLooper = thread.getLooper();
        // HandlerThread 线程的 Handler 对象
        mServiceHandler = new ServiceHandler(mServiceLooper);
    }
}

public class HandlerThread extends Thread {
    Looper mLooper;

    public HandlerThread(String name) {
        super(name);
        mPriority = Process.THREAD_PRIORITY_DEFAULT;
    }

    @Override
    public void run() {
        mTid = Process.myTid();
        Looper.prepare();
        synchronized (this) {
            mLooper = Looper.myLooper();
            notifyAll();
        }
        Process.setThreadPriority(mPriority);
        onLooperPrepared();
        Looper.loop();
        mTid = -1;
    }
}

private final class ServiceHandler extends Handler {
    public ServiceHandler(Looper looper) {
        super(looper);
    }
    @Override
    public void handleMessage(Message msg) {
        // 已经在子线程中执行
        onHandleIntent((Intent)msg.obj);
        stopSelf(msg.arg1);
    }
}
```

在创建了一个 IntentService 对象后，就会有一个新的线程创建出来，并且线程的 Looper 开始轮询，Handler 开始监听。

```java
public abstract class IntentService extends Service {
    @Override
    // service 的生命周期
    public int onStartCommand(@Nullable Intent intent, int flags, int startId) {
        onStart(intent, startId);
        return mRedelivery ? START_REDELIVER_INTENT : START_NOT_STICKY;
    }
    
    @Override
    // Handler 发送消息
    public void onStart(@Nullable Intent intent, int startId) {
        Message msg = mServiceHandler.obtainMessage();
        msg.arg1 = startId;
        msg.obj = intent;
        mServiceHandler.sendMessage(msg);
    }
}
```

