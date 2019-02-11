##### Q1：Android Handler 原理和源码分析？

Android 中通过 Handler，可以将任务转移到 Handler 所在的线程中执行。底层通过 ThreadLocal 保存不同线程的 Looper 对象，然后循环获取 Looper 对象的成员变量 MessageQueue 中的任务，最后交给 Handler 处理，实现了线程的转移。

- ThreadLocal —— 同一个ThreadLocal 对象在不同的线程中可以获取/保存不同的值

  ```java
  /**
   * 获取一个ThreadLocalMap对象，然后将当前ThreadLocal对象当作key来获取值
   */
  public T get() {
      Thread t = Thread.currentThread();
      ThreadLocalMap map = getMap(t);
      if (map != null) {
          ThreadLocalMap.Entry e = map.getEntry(this);
          if (e != null) {
              @SuppressWarnings("unchecked")
              T result = (T)e.value;
              return result;
          }
      }
      return setInitialValue();
  }
  
  /**
   * ThreadLocalMap对象实际上是Thread的一个成员变量
   * 也就是说，每一个Thread内部都有一个ThreadLocalMap成员，它内部存放的是不同的ThreadLocal对象和其对应的值
   */
  ThreadLocalMap getMap(Thread t) {
      return t.threadLocals;
  }
  
  /**
   * 设置ThreadLocal对象在currentThread中的值，如果ThreadLocalMap不存在就创建一个
   */
  public void set(T value) {
      Thread t = Thread.currentThread();
      ThreadLocalMap map = getMap(t);
      if (map != null)
          map.set(this, value);
      else
          createMap(t, value);
  }
  
  /**
   * ThreadLocal实际存放的值是保存在每个Thread的ThreadLocalMap中的
   */
  void createMap(Thread t, T firstValue) {
      t.threadLocals = new ThreadLocalMap(this, firstValue);
  }
  
  static class ThreadLocalMap {
      static class Entry extends WeakReference<ThreadLocal> {
          Object value;
          Entry(ThreadLocal k, Object v) {
              super(k);
              value = v;
          }
      }
      /**
       * 实际存放值的容器
       */
      private Entry[] table;
      /**
       * 通过ThreadLocal hash的取模操作得到当前ThreadLoal对应的值
       */
      private Entry getEntry(ThreadLocal key) {
          int i = key.threadLocalHashCode & (table.length - 1);
          Entry e = table[i];
          if (e != null && e.get() == key)
              return e;
          else
              return getEntryAfterMiss(key, i, e);
      }
      /**
       * 添加元素
       */
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

  总结来说，ThreadLocal 能够在不同线程中保存/获取不同的值，是因为每个 Thread 内有一个保存 ThreadLocal 对象的数组，每次的存取操作都根据当前 ThreadLocal 对象的 hashcode 得到数据在数组中的位置，然后进行处理的。

然后分析下 Handler 的创建、发送消息、处理消息的流程代码。

```java
/**
 * Handler的各种构造函数最后都会调用到这个构造方法
 */
public Handler(Callback callback, boolean async) {
    if (FIND_POTENTIAL_LEAKS) {
        final Class<? extends Handler> klass = getClass();
        if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                (klass.getModifiers() & Modifier.STATIC) == 0) {
            Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                klass.getCanonicalName());
        }
    }
    // 获取当前线程的Looper对象，主线程中已经创建过了，如果是新建线程，需要在创建Handler对象前，调用Looper.prepare()来创建一个当前线程的Looper对象，否则就会抛出下面的异常
    mLooper = Looper.myLooper();
    if (mLooper == null) {
        throw new RuntimeException(
            "Can't create handler inside thread that has not called Looper.prepare()");
    }
    // Looper类有一个用于保存任务的成员变量MessageQueue
    mQueue = mLooper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}
/**
 * 每个线程中保存不同的Looper对象
 */
public static @Nullable Looper myLooper() {
    return sThreadLocal.get();
}
/**
 * Handler各种 post\sendxxx 方法，最后都会调用这个方法，将Message放入MessageQueue中，并设置msg.target是当前Handler对象
 */
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    msg.target = this;
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}
/**
 * 在Looper.loop() 中会调用 msg.target.dispatchMessage 也就是当前Handler.dispatchMessage()处理消息
 */
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

接下来的问题就是 Looper 中如何调用到 Handler.dispatchMessage，MessageQueue 是如何添加 Message 对象的，。

```java
public final class Looper {
    /**
     * 保存Looper对象
     */
    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
    /**
     * 消息队列，本质是一个单链表
     */
    final MessageQueue mQueue;
    /**
     * 当前线程
     */
    final Thread mThread;
    private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
    }
    /**
     * 对于没有Looper对象的线程，需要调用此方法创建
     */
    public static void prepare() {
        prepare(true);
    }
    /**
     * 创建当前线程的Looper对象，并放入Thread.ThreadLocalMap中
     */
    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }
    /**
     * 开启循环取MessageQueue的消息，一般在创建Handler对象后调用
     */
    public static void loop() {
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;
        for (;;) {
            // queue.next()有可能会造成阻塞
            Message msg = queue.next();
            if (msg == null) {
                return;
            }
            try {
                msg.target.dispatchMessage(msg);
            } finally {
                ...
            }
            msg.recycleUnchecked();
        }
    }
}
```

上面说到 MessageQueue.next 方法会造成阻塞，下面分析下 MessageQueue 源码

```java
/**
 * 向MessageQueue添加元素，MessageQueue的底层数据结构是单链表
 */
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
            IllegalStateException e = new IllegalStateException(msg.target + " sending message to a Handler on a dead thread");
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
            msg.next = p;
            prev.next = msg;
        }
        if (needWake) {
            nativeWake(mPtr);
        }
    }
    return true;
}
/**
 * 获取单链表的下一个结点，有可能造成阻塞
 */
Message next() {
    final long ptr = mPtr;
    if (ptr == 0) {
        return null;
    }
    int pendingIdleHandlerCount = -1;
    int nextPollTimeoutMillis = 0;
    // 开启循环取消息
    for (;;) {
        ...
        synchronized (this) {
            // 获取当前事件，用于对比延迟任务的when
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
            if (msg != null && msg.target == null) {
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

Activity.runOnUiThread 和 View.post 等方法，底层都是通过获取主线程的 Handler 来实现的。

##### Q2：Android AsyncTask 源码分析？

AsyncTask 使用时有两个注意点：

1. 一个对象只能执行一次 execute 方法;
2. 同一个 AsyncTask 类的对象，会共享一个任务队列

可以从源码上看到这两个注意点的原因。我们使用 AsyncTask 通常是重写 doInBackground 方法然后调用 execute 方法开始执行任务。所以我们以构造函数开始，然后从 execute 方法作为入口分析源码

```java
/**
 * 返回结果在主线程处理
 */
public AsyncTask() {
    this((Looper) null);
}
/**
 * 返回结果指定在某个handler所在的线程处理
 */
public AsyncTask(@Nullable Handler handler) {
    this(handler != null ? handler.getLooper() : null);
}
/**
 * WorkerRunnable 实现了Callable接口
 * FutureTask 实现Runnable接口，包装了WorkerRunnable对象和一个state状态
 */
public AsyncTask(@Nullable Looper callbackLooper) {
    mHandler = callbackLooper == null || callbackLooper == Looper.getMainLooper()
        ? getMainHandler()
        : new Handler(callbackLooper);
    // 可以看到call()的回调中会开始执行 doInBackground 方法
    // 然后在finally中将result通过postResult通知出去
    mWorker = new WorkerRunnable<Params, Result>() {
        public Result call() throws Exception {
            Result result = null;
            try {
                result = doInBackground(mParams);
            } catch (Throwable tr) {
                mCancelled.set(true);
                throw tr;
            } finally {
                postResult(result);
            }
            return result;
        }
    };
    mFuture = new FutureTask<Result>(mWorker) {
        @Override
        protected void done() {
            try {
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

/**
 * getHandler()获取到的是构造函数中设置的mHandler
 */
private Result postResult(Result result) {
    Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT, new AsyncTaskResult<Result>(this, result));
    message.sendToTarget();
    return result;
}
```

构造函数大致如此，我们可以看到 doInBackground 是在 WorkRunnable 的 call() 回调中执行的。然后看下 AsyncTask 的 execute() 方法。

```java
/**
 * 在默认的Executor上执行任务
 */
public final AsyncTask<Params, Progress, Result> execute(Params... params) {
    return executeOnExecutor(sDefaultExecutor, params);
}
/**
 * 在此方法中会执行 onPreExecute()
 * mStatus的状态判断一个对象只能执行一次 executor 方法
 * 然后调用Executor执行任务，参数赋值到mWorker，然后mWorker被包装到mFuture
 */
public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec， Params... params) {
    if (mStatus != Status.PENDING) {
        switch (mStatus) {
            case RUNNING:
                throw new IllegalStateException("Cannot execute task: the task is already running.");
            case FINISHED:
                throw new IllegalStateException("Cannot execute task: the task has already been executed (a task can be executed only once)");
        }
    }
    mStatus = Status.RUNNING;
    onPreExecute();
    mWorker.mParams = params;
    exec.execute(mFuture);
    return this;
}

private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;
public static final Executor SERIAL_EXECUTOR = new SerialExecutor();
private static class SerialExecutor implements Executor {
    // mTasks是一个保存任务的列表
    // 同一个 AsyncTask 类的对象，会共享一个任务队列
    final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
    Runnable mActive;
    public synchronized void execute(final Runnable r) {
        mTasks.offer(new Runnable() {
            public void run() {
                try {
                    // 实际上调用的是 mFuture 的run方法
                    // 而在mFuture的run方法中会调用包装起来的WorkerRunnable.call()方法
                    r.run();
                } finally {
                    scheduleNext();
                }
            }
        });
        // 开启线程，最后执行WorkerRunnable.call()，doInBackground在子线程中执行
        if (mActive == null) {
            scheduleNext();
        }
    }
    protected synchronized void scheduleNext() {
        if ((mActive = mTasks.poll()) != null) {
            THREAD_POOL_EXECUTOR.execute(mActive);
        }
    }
}
```

##### Q3：Android IntentService 源码分析？

IntentService 继承 Service，通过重写 onHandleIntent() 在子线程中执行任务，为什么 onHandleIntent 会在子线程中执行？执行后 IntentService 会一直存在还是被停止。

```java
public abstract class IntentService extends Service {
    @Override
    public void onCreate() {
        super.onCreate();
        // HandlerThread 是一个包含Looper对象的Thread的子类
        HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
        // start() 会调用 Looper.prepare() 和 loop() 开启轮询
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

