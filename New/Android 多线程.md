# Android 多线程

## Handler 机制

Handler 是 Android 中的消息处理机制，通过 Handler 发送 Message/Runnable 到线程的消息队列（MessageQueue）中，通过和 Handler 绑定的 Looper 对象循环遍历取出 Message/Runnable，将任务转移到 Handler 所在的线程中执行。

### Handler 构造函数

在 Android 中使用 Handler 一般用法是，创建一个 Handler 子类，并重写 handleMessage 方法，根据 Message 对象的 what 属性区分处理不同的消息，或者直接使用 Handler 的 post(Runnable r) 方法执行一段逻辑。

```java
final Handler mHandler = new Handler(Looper.getMainLooper()) {
    @Override
    public void handleMessage(Message msg) {
        switch (msg.what) {
            case xxx:
                ...
                break;
            default:
                super.handleMessage(msg);
        }
    }
};

...
private void methodA(){
    Message message = Message.obtain();
    message.what = xxx
    mHandler.sendMessage(message);
}
private void methodB(){
    mHandler.post(New Runuable());
}
```

Handler 的构造函数

```
final Looper mLooper;
final MessageQueue mQueue;
final Callback mCallback;
final boolean mAsynchronous;

public Handler(@NonNull Looper looper) {
    this(looper, null, false);
}

public Handler(@NonNull Looper looper, @Nullable Callback callback, boolean async) {
    mLooper = looper;
    mQueue = looper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}
```

#### Looper

Looper.getMainLooper() 方法中取出的是在 ActivityThread 静态方法 main 中创建的对象。

```java
//ActivityThread.java
public static void main(String[] args) {
    ···
    Looper.prepareMainLooper();
    ···
    ActivityThread thread = new ActivityThread();
    thread.attach(false, startSeq);

    if (sMainThreadHandler == null) {
        sMainThreadHandler = thread.getHandler();
    }
    ···
    Looper.loop();
    ···
}
```

prepareMainLooper() 创建了主线程的 Looper 对象，并将该对象存放到线程局部变量 sThreadLocal 中

sMainThreadHandler 是 ActivityThread 中成员变量 Handler 的子类。

Looper.loop() 开启了循环遍历。

```java
public static void prepareMainLooper() {
    prepare(false);
    synchronized (Looper.class) {
        if (sMainLooper != null) {
            throw new IllegalStateException("The main Looper has already been prepared.");
        }
        sMainLooper = myLooper();
    }
}

private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));
}

private Looper(boolean quitAllowed) {
	static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
	mQueue = new MessageQueue(quitAllowed);
	mThread = Thread.currentThread();
}

public static @Nullable Looper myLooper() {
    return sThreadLocal.get();
}
```

prepareMainLooper 会判断 sMainLooper 是否有值，如果调用多次，就会抛出异常，所以也就是说主线程的Looper 和 MessageQueue 只会有一个。

同理子线程中调用 Looper.prepare() 时，会调用 prepare(true) 方法，如果多次调用，也会抛出每个线程只能有一个 Looper 的异常。

总结起来就是每个线程中只有一个 Looper 和 MessageQueue。

#### ThreadLocal

同一个 ThreadLocal 对象在不同的线程中可以获取/保存不同的值。

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

总结来说，ThreadLocal 能够在不同线程中保存/获取不同的值，是因为每个 Thread 内有一个保存 ThreadLocalMap，每次的存取操作都根据当前 ThreadLocal 对象的 hashcode 得到数据在数组中的位置，然后进行处理的。

#### MessageQueue

消息队列 MessageQueue 的底层数据结构是单链表。

```java
public final class MessageQueue {
	Message mMessages;
}
```

mMessages 是单链表的头结点。

### Handler 插入消息

Handler 发送消息的各种 post/sendMessage 最终都会调用 sendMessageAtTime

```java
// Handler.java
public final boolean post(@NonNull Runnable r) {
   return  sendMessageDelayed(getPostMessage(r), 0);
}

private static Message getPostMessage(Runnable r) {
    Message m = Message.obtain();
    // post方法发送消息，设置Message.callback对象
    m.callback = r;
    return m;
}

public boolean sendMessageAtTime(@NonNull Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
        Log.w("Looper", e.getMessage(), e);
        return false;
    }
    return enqueueMessage(queue, msg, uptimeMillis);
}

private boolean enqueueMessage(@NonNull MessageQueue queue, @NonNull Message msg,
        long uptimeMillis) {
    msg.target = this;
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}
```

最终通过 enqueueMessage 将 Message 放入 MessageQueue 中，并将当前 Handler 对象设置为 Message 对象的 target（被当前 Handler 消费），mAsynchronous 是在 Handler 构造函数中设置，表示该 Handler 发送的是否是异步消息，默认是 false。

```java
// MessageQueue.java
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
        // 设置当前 Message 正在处理和 when
        msg.markInUse();
        msg.when = when;
        Message p = mMessages;
        boolean needWake;
        if (p == null || when == 0 || when < p.when) {
            // 如果没有头结点，或者执行任务延迟时间 when 是0，或者延迟时间小于头结点的延迟时间
            // 就将当前需要插入的Message设置为头结点
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked;
        } else {
            // 除以上这些情形，通过when属性，遍历单链表，找到合适的位置插入Message
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
```

### Handler 取出消息

ActivityThread 的静态方法 main 中，在完成 Looper 和 sMainThreadHandler 对象创建后，开启了循环遍历，从 MessageQueue 中取出 Message 并消费。

```java
// Looper.java
public static void loop() {
    // 得到当前线程的Looper对象
    final Looper me = myLooper();
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }
    me.mInLoop = true;
    
    // 得到当前线程的MessageQueue对象
    final MessageQueue queue = me.mQueue;
    ...
    for (;;) {
        // 不断从当前线程的MessageQueue中取出Message，当MessageQueue没有元素时，方法阻塞
        Message msg = queue.next();
        if (msg == null) {
            return;
        }
        ...
        try {
            // Message.target是发送消息的Handler，调用它的dispatchMessage方法
            msg.target.dispatchMessage(msg);
            ...
        } catch (Exception exception) {
            ...
        } finally {
            ...
        }
        ...
        // 回收Message对象
        msg.recycleUnchecked();
    }
}

// Handler.java
public void dispatchMessage(@NonNull Message msg) {
    if (msg.callback != null) {
        // msg.callback是一个Runnable对象，是Handler.post方式传递进来的参数
        handleCallback(msg);
    } else {
        // Handler构造函数中mCallback不为空，由mCallback处理消息
        // 如果返回了true，表示已经处理，如果返回false，最终由 handleMessage 重写方法处理
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        handleMessage(msg);
    }
}
```

首先还是判断了当前线程是否有 Looper，然后得到当前线程的 MessageQueue。

接下来通过一个死循环，不断调用 MessageQueue.next() 取出 Message。如果 MessageQueue 中没有消息时，会阻塞在 MessageQueue.next 中的 nativePollOnce() 方法里，导致当前线程挂起。

拿到 Message 后，会调用发送消息时用到的 Handler 的 dispatchMessage 方法。

```java
Message next() {
    ...
    int pendingIdleHandlerCount = -1;
    int nextPollTimeoutMillis = 0;
    // 开启循环取消息
    for (;;) {
        ...
        // 消息被插入或者后面逻辑中设置的nextPollTimeoutMillis时间到，就被唤醒，否则会阻塞在这里
        nativePollOnce(ptr, nextPollTimeoutMillis);
        synchronized (this) {
            // 获取当前事件，用于对比延迟任务的when
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
            if (msg != null && msg.target == null) {
                // msg.target=null的消息是屏障消息
                // 用于标识当前只从队列中取异步消息，这种机制叫同步屏障
                do {
                    prevMsg = msg;
                    msg = msg.next;
                    //isAsynchronous()返回true是异步消息
                } while (msg != null && !msg.isAsynchronous());
            }
            //如果没有屏障消息，msg指向头结点，如果有屏障消息，msg指向第一个异步消息
            if (msg != null) {
                if (now < msg.when) {
                    // 如果还没到处理时间，设置nativePollOnce方法的阻塞时间参数
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    //到处理时间了，就从链表中移除，返回这个消息
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
                // 如果没有msg消息，就一直休眠，等待被唤醒
                nextPollTimeoutMillis = -1;
            }
            // 如果messagequeue退出标示位是ture，就返回null
            if (mQuitting) {
                dispose();
                return null;
            }
            // IdleHanlder接口：当MessageQueue中无可处理的Message，线程将要进入堵塞，以等待更多消息，在这之前回调这个接口的对象
            // mIdleHandlers 保存还未执行的IdleHanlder对象
            // mPendingIdleHandlers 表示将要执行的IdleHanlder对象
            private final ArrayList<IdleHandler> mIdleHandlers = new ArrayList<IdleHandler>();
            private IdleHandler[] mPendingIdleHandlers;
            // 将要执行的IdleHanlder对象为空 并且 messagequeue没消息或者消息执行时间未到
            if (pendingIdleHandlerCount < 0
                    && (mMessages == null || now < mMessages.when)) {
                // 先赋值pendingIdleHandlerCount，后面会把mIdleHandlers数据复制到mPendingIdleHandlers中
                pendingIdleHandlerCount = mIdleHandlers.size();
            }
            if (pendingIdleHandlerCount <= 0) {
                //如果没有将要执行的IdleHanlder对象，阻塞线程
                mBlocked = true;
                continue;
            }
            if (mPendingIdleHandlers == null) {
                mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
            }
            // 把mIdleHandlers数据复制到mPendingIdleHandlers中
            mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
            
            // 遍历mPendingIdleHandlers
            for (int i = 0; i < pendingIdleHandlerCount; i++) {
                final IdleHandler idler = mPendingIdleHandlers[i];
                mPendingIdleHandlers[i] = null; 
                boolean keep = false;
                try {
                    keep = idler.queueIdle();
                } catch (Throwable t) {
                    Log.wtf(TAG, "IdleHandler threw exception", t);
                }
                if (!keep) {
                    synchronized (this) {
                        mIdleHandlers.remove(idler);
                    }
                }
            }
            // 重置数量
            pendingIdleHandlerCount = 0;
            nextPollTimeoutMillis = 0;
        }
    }
}
```

#### Handler 同步屏障

调用 requestLayout() 方法之后，并不会马上开始进行绘制任务，而是会给主线程设置一个同步屏障，并设置 ASYNC 信号监听。当 ASYNC 信号的到来，会发送一个异步消息到主线程 Handler，执行我们上一步设置的绘制监听任务，并移除同步屏障。

这样的好处是：

- 保证在ASYNC信号到来之时，绘制任务可以被及时执行，不会造成界面卡顿。

这样的代价：

- 我们的同步消息最多可能被延迟一帧的时间，也就是16ms，才会被执行
- 主线程 Looper 造成过大的压力，在 VSYNC 信号到来之时，才集中处理所有消息

Handler 发送的消息分为普通消息、屏障消息、异步消息，一旦在 Looper 处理消息 MessageQueue,next 方法中遇到屏障消息，那么就不再处理普通的消息，而仅仅处理异步的消息。不再使用屏障后，需要撤销屏障，不然就再也执行不到普通消息了。

- 屏障消息：Message.target = null 的消息
- 异步消息：Message.target != null 并且 Message.isAsynchronous() 返回 true 的消息
- 普通消息：Message.target != null 并且 Message.isAsynchronous() 返回 false 的消息



在 Message 被插入到 MessageQueue 中，会判断 target 属性不为空的：

```java
// MessageQueue.java
boolean enqueueMessage(Message msg, long when) {
  // Hanlder不允许为空
  if (msg.target == null) {
      throw new IllegalArgumentException("Message must have a target.");
  }
  ...
}
```

所以 MessageQueue 提供了 postSyncBarrier() 方法在插入 target==null 的屏障消息：

```java
// MessageQueue.java
private int postSyncBarrier(long when) {
    synchronized (this) {
        final int token = mNextBarrierToken++;
        final Message msg = Message.obtain();
        msg.markInUse();
        msg.when = when;
        msg.arg1 = token;
        Message prev = null;
        Message p = mMessages;
        // 把当前队列中when之前的Message全部放到屏障消息前，使得他们全部执行
        if (when != 0) {
            while (p != null && p.when <= when) {
                prev = p;
                p = p.next;
            }
        }
        // 插入同步屏障
        if (prev != null) {
            msg.next = p;
            prev.next = msg;
        } else {
            msg.next = p;
            mMessages = msg;
        }
        return token;
    }
}
```

如果遇到同步屏障，那么会循环遍历整个链表找到标记为异步消息，其他的消息会直接忽视，这样异步消息就会提前被执行了。

同步屏障不会自动移除，使用完成之后需要手动进行移除，不然会造成同步消息无法被处理。

```java
// MessageQueue.java
public void removeSyncBarrier(int token) {
    synchronized (this) {
        Message prev = null;
        Message p = mMessages;
        while (p != null && (p.target != null || p.arg1 != token)) {
            prev = p;
            p = p.next;
        }
        if (p == null) {
            throw new IllegalStateException("The specified message queue synchronization "
                    + " barrier token has not been posted or has already been removed.");
        }
        final boolean needWake;
        if (prev != null) {
            prev.next = p.next;
            needWake = false;
        } else {
            mMessages = p.next;
            needWake = mMessages == null || mMessages.target != null;
        }
        p.recycleUnchecked();
        // We can assume mPtr != 0 when mQuitting is false.
        if (needWake && !mQuitting) {
            nativeWake(mPtr);
        }
    }
}
```

> 总结
>
> Handler 创建的时候会采用当前线程的 Looper 来构造消息循环系统，Looper 在哪个线程创建，就跟哪个线程绑定，这个绑定关系是通过 ThreadLocal 成员变量完成的。ThreadLocal 可以在不同的线程中互不干扰的存储并提供数据。
>
> 线程默认是没有 Looper 的，如果需要使用 Handler，就必须为线程创建 Looper。所以在 ActivityThread main 静态方法中，先后执行了：
>
> 1. Looper.prepareMainLooper()：创建主线程的 Looper 对象，Looper 内部创建了 MessageQueue 队列
> 2. sMainThreadHandler = thread.getHandler()：ActivityThread 内部创建了 Handler 对象
> 3. Looper.loop()：开启循环遍历
>

## Android HandlerThread 原理分析

HandlerThread 是 Thread 的子类，在内部封装了 Looper 和 Handler

```java
public class HandlerThread extends Thread {
    Looper mLooper;
    private @Nullable Handler mHandler;
    
    @Override
    public void run() {
        Looper.prepare();
        synchronized (this) {
            mLooper = Looper.myLooper();
            notifyAll();
        }
        onLooperPrepared();
        Looper.loop();
    }
    
    public Handler getThreadHandler() {
        if (mHandler == null) {
            mHandler = new Handler(getLooper());
        }
        return mHandler;
    }
}
```

在线程启动的 run 方法中，初始化了 Looper 对象并开启了循环。通过 getThreadHandler 可以获得 Handler 对象。

## Android AsyncTask 原理分析

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



## Android IntentService 原理分析

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



[Handler同步屏障 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/361508626)

[Android进阶之Handle和Looper消息机制原理和源码分析（不走弯路） (qq.com)](https://mp.weixin.qq.com/s?__biz=Mzk0NDE3MjM1Ng==&mid=2247484566&idx=1&sn=e8447f019bd6b491e48cefd79dfcb26d&chksm=c329f8bdf45e71ab53c401775084bfb8cf3be82b7cdee552da1e0b65e8f4fed2d6511f348fb8)

[Handler的内功心法，值得拥有！ (qq.com)](https://mp.weixin.qq.com/s?__biz=MzA5MzI3NjE2MA==&mid=2650253465&idx=1&sn=e2eaf4877df74098f0df7462a7abe0d2&chksm=886359f6bf14d0e0bf3a5e0deb471fc749f63bea898882d3d18666a5e9c4f1cd849083d2386b)

[关于Handler 的这 15 个问题，你都清楚吗？ (qq.com)](https://mp.weixin.qq.com/s?__biz=MzAxMTI4MTkwNQ==&mid=2650832447&idx=1&sn=52356ffeb588ae1a27a92a97e171c278&chksm=80b7aaa1b7c023b7356aab3d1848629b84865403fa6ce48fc6220068619f348924823eab9a7e)

[从源码角度理解如何获取 View 的宽高 (qq.com)](https://mp.weixin.qq.com/s/aA-9UTlebdj5-K4z30f_4g)

