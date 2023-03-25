# Android 多线程

## Handler 机制

Handler 是 Android 中的消息处理机制，在不同的线程通过 Handler 发送 Message/Runnable 到 Handler 所在的线程，然后在该线程的消息队列（MessageQueue）中循环遍历取出 Message/Runnable 执行。

### 核心类

#### Looper

Looper 的构造函数是由 private 修饰，所以在 ActivityThread.main() 中，是通过 prepareMainLooper() 创建了 主线程的 Looper 对象，实际上其内部还是通过 prepare() 创建 Looper 对象。整理流程是：

- prepare() 内部通过构造函数创建 Looper(quitAllowe) 对象
- Looper 构造函数中创建 MessageQueue 对象(mQueue)并获取当前所在线程(mThread)
- sThreadLocal 保存上一步创建的当前线程的 Looper 对象

```java
//Looper.java
public static void prepareMainLooper() {
    prepare(false);
    synchronized (Looper.class) {
        //多次调用会抛出异常，主线程的 Looper 和 MessageQueue 只会有一个
        if (sMainLooper != null) {
            throw new IllegalStateException("The main Looper has already been prepared.");
        }
        sMainLooper = myLooper();
    }
}

private static void prepare(boolean quitAllowed) {
    ////多次调用会抛出异常，一个线程的 Looper 和 MessageQueue 只会有一个
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));
}

public final class Looper {
    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
    private static Looper sMainLooper;
    final MessageQueue mQueue;
    final Thread mThread;
    
    private Looper(boolean quitAllowed) {
    	mQueue = new MessageQueue(quitAllowed);
    	mThread = Thread.currentThread();
	}
    
     public static Looper myLooper() {
    	 return sThreadLocal.get();
 	}
    
    public static MessageQueue myQueue() {
        return myLooper().mQueue;
    }
    
    public static Looper getMainLooper() {
   		synchronized (Looper.class) {
        	return sMainLooper;
    	}
	}
}
```

然后调用 Looper.loop() 就会开启循环，不断从 MessageQueue 中取出消息，然后处理消息。

```java
//Looper.java
public static void loop() {
    //当前线程的Looper对象
    final Looper me = myLooper();
    me.mInLoop = true;
    //当前线程的MessageQueue对象
    final MessageQueue queue = me.mQueue;
    ...
    for (;;) {
        //从当前线程的MessageQueue中取出Message，当MessageQueue没有元素时，方法阻塞
        Message msg = queue.next();
        if (msg == null) {
            return;
        }
        ...
        try {
            //Message.target是发送消息的Handler，调用它的dispatchMessage方法
            msg.target.dispatchMessage(msg);
        }
        ...
        // 回收Message对象
        msg.recycleUnchecked();
    }
}
```

#### ThreadLocal

ThreadLocal 用于保存/获取同一个引用在不同的线程对应不同的实际对象。实现方式是：

- 获取当前线程对象， Thread.java 内包含一个 ThreadLocal.ThreadLocalMap 成员变量
- ThreadLocalMap 内部包含 Entry[] 数组，用于保存不同线程之间独有的对象
- Entry 相当于一个 Map，key 是 ThreadLocal 对象，value 是 需要保存的对象

```java
//ThreadLocal.java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}

void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}

public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}

//set/get方法都是获取thread.threadLocals对象
//每一个Thread内部都有一个ThreadLocalMap成员，它内部存放的是不同的ThreadLocal对象和其对应的值
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}

//ThreadLocalMap.kava
static class ThreadLocalMap {
	ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
		table = new Entry[INITIAL_CAPACITY];
		int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
		table[i] = new Entry(firstKey, firstValue);
		size = 1;
		setThreshold(INITIAL_CAPACITY);
	}
	
	static class Entry extends WeakReference<ThreadLocal<?>> {
		Object value;
		Entry(ThreadLocal<?> k, Object v) {
			super(k);
			value = v;
		}
	}
}
```

#### MessageQueue

在 Looper 的构造函数中创建了 Looper 对应的 MessageQueue，用于 loop() 时从中获取 Message。

```java
public final class MessageQueue {
	//内部Message以单链表形式存在，保存单链表头
	Message mMessages;
	//消息队列空闲时执行特定的操作，不会阻塞消息队列的处理
	private final ArrayList<IdleHandler> mIdleHandlers = new ArrayList<IdleHandler>();
	private IdleHandler[] mPendingIdleHandlers;
}

public final class Message implements Parcelable {
	//消息的执行时间
	public long when;
	//下一个消息的引用
	Message next;
	
	//接收消息的 Handler 对象
	Handler target;
	//消息的类型
	public int what;
	//消息的数据对象，可以存储任意类型的数据
	public Object obj;
	//消息的回调接口
	Runnable callback;
    
    public boolean isAsynchronous() {
        return (flags & FLAG_ASYNCHRONOUS) != 0;
	}
    
    public void setAsynchronous(boolean async) {
        if (async) {
            flags |= FLAG_ASYNCHRONOUS;
        } else {
            flags &= ~FLAG_ASYNCHRONOUS;
        }
	}
}
```

在后文中会将 Message 分为三种类型：

- 同步消息：target=Handler，Message.setAsynchronous(false) 的消息
- 异步消息：target=Handler，Message.setAsynchronous(true) 的消息
- 同步屏障消息：target=null

#### Handler

Handler 主要作用是将消息发送到 MessageQueue 中，再通过 Looper 循环读取消息并处理。Handler 可以在子线程中创建并与主线程进行通信，也可以在主线程中创建并处理异步任务等操作。需要注意的是，创建 Handler 时，必须保证当前线程的 Looper 实例已经创建。

```java
public class Handler {
	final Looper mLooper;
	final MessageQueue mQueue;
	final Callback mCallback;
    //在发送Message时设置是否是异步消息
	final boolean mAsynchronous;
	
	public Handler() {
		this(null, false);
	}
	
	public Handler(Callback callback) {
		this(callback, false);
	}
	
	public Handler(@Nullable Callback callback, boolean async) {
		mLooper = Looper.myLooper();
		if (mLooper == null) {
			throw new RuntimeException(Can't create ... has not called Looper.prepare()");
		}
		mQueue = mLooper.mQueue;
		mCallback = callback;
		mAsynchronous = async;
	}
	
	public Handler(@NonNull Looper looper) {
		this(looper, null, false);
	}
	
	public Handler(@NonNull Looper looper, @Nullable Callback callback, boolean async) {
		mLooper = looper;
		mQueue = looper.mQueue;
		mCallback = callback;
		mAsynchronous = async;
	}
}
```

### 发送消息

通过 Handler 发送消息的若干个 post/sendMessage 方法最终都会调用 enqueueMessage()

```java
public final boolean sendEmptyMessage(int what) {
    return sendEmptyMessageDelayed(what, 0);
}

public final boolean sendEmptyMessageDelayed(int what, long delayMillis) {
    Message msg = Message.obtain();
    msg.what = what;
    return sendMessageDelayed(msg, delayMillis);
}

public final boolean sendMessageDelayed(Message msg, long delayMillis) {
    if (delayMillis < 0) {
        delayMillis = 0;
    }
    return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
}

public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        return false;
    }
    return enqueueMessage(queue, msg, uptimeMillis);
}

public final boolean sendMessageAtFrontOfQueue(Message msg) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        return false;
    }
    return enqueueMessage(queue, msg, 0);
}

public final boolean post(Runnable r) {
   return  sendMessageDelayed(getPostMessage(r), 0);
}

public final boolean postAtFrontOfQueue(Runnable r) {
    return sendMessageAtFrontOfQueue(getPostMessage(r));
}

private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    msg.target = this;
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}
```

在 Handler.enqueueMessage() 中会设置 Message.target 为当前 Handler 实例，并设置 Message 是异步消息还是同步消息，最终调用 MessageQueue.enqueueMessage() 将 Message 加入队列。

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
            msg.recycle();
            return false;
        }
        // 设置Message正在处理
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
           //判断是否需要唤醒
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            Message prev;
            // 除以上这些情形，通过when属性，遍历单链表，找到合适的位置插入Message
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
        //needWake=true，唤醒
        if (needWake) {
            nativeWake(mPtr);
        }
    }
    return true;
}
```

#### 同步屏障消息

MessageQueue 通过以下方式发送同步屏障消息，同步屏障消息不会自动移除，使用完成之后需要手动进行移除，不然会造成同步消息无法被处理：

- 屏障消息和普通消息区别在于屏幕没有target
- 屏障和普通消息一样可以根据时间来插入到消息队列中的适当位置，并且只会挡住它后面的同步消息的分发
- postSyncBarrie r会返回一个token，利用这个token可以撤销屏障
- 插入普通消息会唤醒消息队列，但插入同步屏障消息不会

```java
//MessageQueue.java
public int postSyncBarrier() {
    return postSyncBarrier(SystemClock.uptimeMillis());
}

//发送同步屏障消息
private int postSyncBarrier(long when) {
    synchronized (this) {
        //这个token在移除屏障时会使用到
        final int token = mNextBarrierToken++;
        final Message msg = Message.obtain();
        msg.markInUse();
        msg.when = when;
        msg.arg1 = token;

        //在屏障的时间到来之前的普通消息，不会被屏蔽
        Message prev = null;
        Message p = mMessages;
        if (when != 0) {
            while (p != null && p.when <= when) {
                prev = p;
                p = p.next;
            }
        }

         //插入到单链表中
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

//移除同步屏障消息
public void removeSyncBarrier(int token) {
    // Remove a sync barrier token from the queue.
    // If the queue is no longer stalled by a barrier then wake it.
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
        // If the loop is quitting then it is already awake.
        // We can assume mPtr != 0 when mQuitting is false.
        if (needWake && !mQuitting) {
            nativeWake(mPtr);
        }
    }
}
```

### 选取消息

在 Looper.loop() 中，通过 MessageQueue.next() 来从队列中选取 Message：

- 如果队列头部是同步屏障消息，那么只选取队列中的异步消息
- 如果队列头部不是同步屏障消息，按照 Message.when 设置的时间选取 Messaage
- 如果队列第一个 Message 设置的处理时间 when 都没有到达，就会在 native 层面阻塞
- 在线程将要进入堵塞之前，会处理保留在 MessageQueue.mIdleHandlers 中的事务

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
                //如果没有异步消息就一直休眠，等待被唤醒
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

### 处理消息

在 Looper.loop() 中，通过 Handler.dispatchMessage(msg)  按照以下的优先级来处理 Message：

- Message.callback： Runnable 实例，一般是 Handler.post() 传入
- Handler.mCallback：Callback 实例，调用 handleMessage 处理 Message
- Handler.handleMessage：

```java
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

## HandlerThread

由于 Handler 必须要在包含了 Looper 的线程中使用，尤其在子线程中创建 Handler，需要写模板代码：

```java
class LooperThread extends Thread {
	public Handler mHandler;
	public void run() {
		Looper.prepare();
		mHandler = new Handler() {
			public void handleMessage(Message msg) {
				// 这里处理消息
			}
		};
		Looper.loop();
	}
}
```

HandlerThread 是 Thread 的子类，在内部封装了 Looper 和 Handler，直接使用这个 Thread 创建 Handler。在线程启动的 run 方法中，初始化了 Looper 对象并开启了循环。通过 getThreadHandler 可以获得 Handler 对象。

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

## AsyncTask

AsyncTask 是 Android 提供的一种异步处理机制，通常是重写 doInBackground 方法然后调用 execute 方法开始执行任务，其内部实现是基于线程池和 Handler 机制。通过将异步任务放入线程池中进行处理，并通过 Handler 机制将处理结果返回到 UI 线程中更新界面。需要注意的是：

- 一个对象只能执行一次 execute 方法;

- 同一个 AsyncTask 类的对象，会共享一个任务队列

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

 doInBackground 是在 WorkRunnable 的 call() 回调中执行的，然后调调用 execute()：

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

## IntentService

IntentService 通过重写 onHandleIntent() 在子线程中执行任务。它可以处理一个任务队列，每个任务通过 Intent 进行传递，以避免阻塞主线程且方便执行高延迟的异步任务，同时也避免因为异步任务执行完成后忘记关闭服务而导致的内存泄露问题。 IntentService 内部实现的主要原理如下：

- IntentService 内部维护了 一个子线程 HandlerThread

- IntentService 将每个发送过来的 Intent 压入队列中去

- 如果工作线程正在执行任务，则将剩下的任务按照先进先出的原则存入队列等待处理

- 当工作线程空闲的时候，把队列中最先进来的 Intent 取出并递交给工作线程进行处理

- 如果任务处理完毕，则检测队列中是否还有未处理的任务，有的话继续进行处理，否则调用 stopSelf() 方法停止服务

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

## ThreadPoolExecutor

### 核心参数

ThreadPoolExecutor 是 Java 中常用的线程池实现类，用于管理线程池中线程的创建、销毁和使用。为了方便开发者创建线程池，Executors 提供了四个静态方法用于创建不同特点的线程池：

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

线程池是通过 ThreadPoolExecutor 提供的一系列参数来配置创建的：

```java
public class ThreadPoolExecutor extends AbstractExecutorService {
	public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime, //时间单位
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler rejectedHandler) {
		if (corePoolSize < 0 ||
			maximumPoolSize <= 0 ||
			maximumPoolSize < corePoolSize ||
			keepAliveTime < 0){
				throw new IllegalArgumentException();
		}
		if (workQueue == null || threadFactory == null || handler == null){
			throw new NullPointerException();
		}
		//核心线程数
		this.corePoolSize = corePoolSize;
        
         //阻塞队列，一般可以选择：
         //ArrayBlockingQueue：基于数组的有界阻塞队列
         //LinkedBlockingQueue：基于链表的阻塞队列，不存在最大值限制
         //SynchronousQueue：不存储元素的阻塞队列，每个插入操作必须等上一个元素被移除
         this.workQueue = workQueue;
        
         //最大线程数
		this.maximumPoolSize = maximumPoolSize;
        
         //keepAliveTime：线程超时时间
         //unit：时间单位，常用 MILLISECOND SECONDS MINUTES HOURS DAYS
		this.keepAliveTime = unit.toNanos(keepAliveTime);
        
         //创建新的线程方式
		this.threadFactory = threadFactory;
        
         //任务拒绝处理器
         //AbortPolicy：丢弃任务，并抛出运行时异常
         //CallerRunsPolicy：只用调用者所在的线程来处理任务
         //DiscardOldestPolicy：丢弃阻塞队列中最近的任务，然后执行当前任务
         //DiscardPolicy：直接丢弃任务，不处理
		this.handler = rejectedHandler;
	}
}
```

ThreadPoolExecutor 中常用的成员变量和常量：

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

### 工作原理

ThreadPoolExecutor 是通过调用 execute() 处理任务，首先会读取 ctl 变量的值来获取当前线程池状态和工作线程数，然后根据当前工作线程数，判断是否可以添加新的工作线程来执行任务：

- 当前线程数＜corePoolSize，创建线程

- 当前线程数＝corePoolSize，放入阻塞队列中排队

- 当前线程数＝corePoolSize，阻塞队列已满，创建新的线程

- corePoolSize＜当前线程数＜maximumPoolSize，阻塞队列未满，放入队列，阻塞队列已满，创建新的线程

- 当前线程数＝maximumPoolSize，阻塞队列已满，执行 rejectedExecutionHandler

- 调用 shutdown() 后，等待 workQueue  任务执行完再关闭，如果此时有新的任务，执行 rejectedExecutionHandler

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

addWorker() 方法完成了线程池主要的创建线程和执行任务的功能：

- 检查线程池状态，如果线程池状态不为 RUNNING，则不再添加新的线程，直接返回
- 根据工作线程状态 (workerCount < corePoolSize 或 workerCount < maximumPoolSize) 判断线程池是否需要创建新的工作线程
- 通过 workerThreadFactory 创建新的工作线程，同时使 workerCount++
- 启动新的工作线程，线程会立即执行任务 (如果当前任务队列不为空)，并将 workerAdded 标志置为 true
- 在锁定的情况下重新检查线程池状态和工作线程数，如果不满足满足创建新线程的条件，则进行回退操作(workers 集合移除新增工作线程、workerCount--)

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

runWorker() 是用于执行任务的工作线程执行体：

- 调用工作线程的 run() 方法，该方法会被在一个循环中执行
- 判断工作线程是否已经被加入到 workers 集合中，如果未加入或已被移除，则结束循环，执行 workerDone() 方法处理线程完成或者失败后的善后工作
- 从线程池任务队列中取出一个新的任务进行执行，如果任务队列为空，则休眠一段时间
- 如果取出的任务不为 null，用 beforeExecute() 对其进行预处理，如果预处理成功则调用任务的 run() 方法进行执行
- 执行完任务后，调用 afterExecute() 方法对任务进行善后处理，并通过 done() 方法将该任务的执行结果传递给调用者(CallerRunsPolicy、DiscardPolicy 或 AbortPolicy)
- 若任务执行过程中出现异常，则调用 "afterExecute" 方法对任务进行异常处理，并将异常传递给 done() 方法

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



[Handler同步屏障 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/361508626)

[Android进阶之Handle和Looper消息机制原理和源码分析（不走弯路） (qq.com)](https://mp.weixin.qq.com/s?__biz=Mzk0NDE3MjM1Ng==&mid=2247484566&idx=1&sn=e8447f019bd6b491e48cefd79dfcb26d&chksm=c329f8bdf45e71ab53c401775084bfb8cf3be82b7cdee552da1e0b65e8f4fed2d6511f348fb8)

[Handler的内功心法，值得拥有！ (qq.com)](https://mp.weixin.qq.com/s?__biz=MzA5MzI3NjE2MA==&mid=2650253465&idx=1&sn=e2eaf4877df74098f0df7462a7abe0d2&chksm=886359f6bf14d0e0bf3a5e0deb471fc749f63bea898882d3d18666a5e9c4f1cd849083d2386b)

[关于Handler 的这 15 个问题，你都清楚吗？ (qq.com)](https://mp.weixin.qq.com/s?__biz=MzAxMTI4MTkwNQ==&mid=2650832447&idx=1&sn=52356ffeb588ae1a27a92a97e171c278&chksm=80b7aaa1b7c023b7356aab3d1848629b84865403fa6ce48fc6220068619f348924823eab9a7e)

