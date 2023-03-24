# View 刷新流程

## Android 屏幕刷新

### 三重缓冲和 VSync（垂直同步）

CPU：主要用于计算数据，在 Android 中主要进行 Surface 的计算过程，起着生产者的作用。

GPU：主要用于画面的渲染，将 CPU 计算好的 Surface 数据合成到 buffer 中，让显示器进行读取，起着消费者的作用。

帧率：GPU 在1秒内可以渲染多少帧到 buffer 中，单位是 fps。

屏幕刷新率：屏幕在1秒内从 buffer 中取数据的次数，单位为Hz。

![image-20230320222207964](C:\Users\24594\AppData\Roaming\Typora\typora-user-images\image-20230320222207964.png)

画面撕裂的问题，本质上是帧率和屏幕刷新率不一致导致的，在 Android 4.1 之前，使用了双缓冲来解决画面撕裂的问题：

- GPU 写入的缓存是：Back Buffer
- 屏幕刷新率使用的缓存是：Frame Buffer

当屏幕刷新时，Frame Buffer 不会发生变化，GPU 持续写入 Back Buffer，然后当屏幕刷新完毕，下一帧刷新之前，通过交换 Buffer 实现帧数据的切换，这个时间点硬件屏幕会发出一个脉冲信号，这个信号就是 VSync 信号。在没有 VSync 信号之前，每次 CPU 计算数据的过程都会等待 GPU 将上一个 Surface 数据渲染后才会进行，这就导致当有一次 GPU 合成过程耗时太多，就会影响接下来的帧率，为了进一步优化渲染性能，系统在收到 VSyns 信号后，会马上进行 CPU 的计算和 GPU 的 buffer 写入。 这样就可以让 CPU 和 GPU  有个完整的16.6ms处理过程。最大限度的减少 jank 的发生。

![image-20230320223953663](C:\Users\24594\AppData\Roaming\Typora\typora-user-images\image-20230320223953663.png)

如果主线程做了一些相对复杂耗时逻辑，导致 CPU 和 GPU 的处理时间超过16.6ms，由于此时 back buffer 写入的是B帧数据，在交换 buffer 前不能被覆盖，而 frame buffer 被 Display 用来做刷新用，所以在B帧写入 back buffer完成到下一个 VSync 信号到来之前两个 buffer 都被占用了，CPU 无法继续绘制，这段时间就会被空着， 于是又出现了三重缓存。在第一个 VSync 信号来时，虽然 back buffer 以及 frame buffer 都被占用了，CPU 此时会启用第三个 Buffer，避免了 CPU 的空闲状态。

![image-20230320224413617](C:\Users\24594\AppData\Roaming\Typora\typora-user-images\image-20230320224413617.png)

### Choreographer

上一节提到 VSync 的最重要的两个功能：

- 标记前后台 Buffer 的切换时机
- CPU 和 GPU 重新开始计算

在 Android 的源码层面是通过 Choreographer 来实现的。[Window](Window.md) 的最后提到 ViewRootImpl 通过 setView() 将 View 添加到 Window 中，其中调用了 requestLayout() 。

```java
ViewRootImpl.java
public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
        checkThread();
        mLayoutRequested = true;
        scheduleTraversals();
    }
}

void scheduleTraversals() {
    //mTraversalScheduled保证不会重复执行，在doTraversal()中重置为false
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        //Handler发送同步屏障消息，保证异步绘制消息优先执行
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        //mChoreographer发送一个Runnable，View的measure、layout、draw等流程都在doTraversal中
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        notifyRendererOfFramePending();
        pokeDrawLockIfNeeded();
    }
}

final class TraversalRunnable implements Runnable {
    @Override
    public void run() {
        doTraversal();
    }
}
final TraversalRunnable mTraversalRunnable = new TraversalRunnable();

void doTraversal() {
    //在scheduleTraversals()中设为true
    if (mTraversalScheduled) {
        mTraversalScheduled = false;
        //Handler移除同步屏障消息
        mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);
        //开启View的测量布局绘制流程
        performTraversals();
    }
}
```

接下来就是通过 mChoreographer.postCallback() 来监听 VSyns 信号，然后回调 Runnable 类型的参数。需要注意的是在最终调用到 postCallbackDelayedInternal() 时，参数 callbackType=Choreographer.CALLBACK_TRAVERSAL，参数token=null。

```java
Choreographer.java
public void postCallback(int callbackType, Runnable action, Object token) {
    ...
    postCallbackDelayedInternal(callbackType, action, token, 0);
}

private void postCallbackDelayedInternal(int callbackType,
        Object action, Object token, long delayMillis) {
    synchronized (mLock) {
        final long now = SystemClock.uptimeMillis();
        final long dueTime = now + delayMillis;
        ////将Runnable封装为CallbackRecord对象后，存到mCallbackQueues数组中
        mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token);
        if (dueTime <= now) {
            scheduleFrameLocked(now);
        } else {
            Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_CALLBACK, action);
            msg.arg1 = callbackType;
            msg.setAsynchronous(true);
            mHandler.sendMessageAtTime(msg, dueTime);
        }
    }
}

private void scheduleFrameLocked(long now) {
    if (!mFrameScheduled) {
        mFrameScheduled = true;
        if (USE_VSYNC) {
            //使用了VSYNC信号，如果是在主线程上，则直接调用scheduleVsyncLocked
            if (isRunningOnLooperThreadLocked()) {
                scheduleVsyncLocked();
            } else {
                Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_VSYNC);
                msg.setAsynchronous(true);
                mHandler.sendMessageAtFrontOfQueue(msg);
            }
        }
        ...
    }
}

//对于延时操作或者子线程，进行处理，最终都会回归总流程
private final class FrameHandler extends Handler {
    public FrameHandler(Looper looper) {
        super(looper);
    }
    @Override
    public void handleMessage(Message msg) {
        switch (msg.what) {
            case MSG_DO_FRAME:
                doFrame(System.nanoTime(), 0, new DisplayEventReceiver.VsyncEventData());
                break;
            case MSG_DO_SCHEDULE_VSYNC:
                doScheduleVsync();
                break;
            case MSG_DO_SCHEDULE_CALLBACK:
                doScheduleCallback(msg.arg1);
                break;
        }
    }
}

private void scheduleVsyncLocked() {
    ...
    //mDisplayEventReceiver是在Choreographer的构造函数中创建的
    //是FrameDisplayEventReceiver的类对象
    mDisplayEventReceiver.scheduleVsync();
}

public void scheduleVsync() {
    //native层注册VSync信号监听，当监听后会调用onVsync回调
    nativeScheduleVsync(mReceiverPtr);
}

private final class FrameDisplayEventReceiver extends DisplayEventReceiver
        implements Runnable {
    @Override
    public void onVsync(long timestampNanos, long physicalDisplayId, int frame,
            VsyncEventData vsyncEventData) {
        try {
            //监听到VSync信号，发送一个异步消息，因为有同步屏障消息的存在，这个异步消息优先执行
            Message msg = Message.obtain(mHandler, this);
            msg.setAsynchronous(true);
            mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
    }
    @Override
    public void run() {
        mHavePendingVsync = false;
        //Message的Runnable，指定doFrame方法
        doFrame(mTimestampNanos, mFrame, mLastVsyncEventData);
    }
}

void doFrame(long frameTimeNanos, int frame) {
    ...
    try {
        //处理输入事件
        doCallbacks(Choreographer.CALLBACK_INPUT, frameTimeNanos);
        //处理动画事件
        doCallbacks(Choreographer.CALLBACK_ANIMATION, frameTimeNanos);
        //处理CALLBACK_TRAVERSAL，三大绘制流程
        doCallbacks(Choreographer.CALLBACK_TRAVERSAL, frameTimeNanos);
    } 
    ...
}

void doCallbacks(int callbackType, long frameTimeNanos) {
    //在postCallbackDelayedInternal中封装的CallbackRecord对象，依次执行Runnable
    //最终调会到ViewRootImpl.performTraversals()
    for (CallbackRecord c = callbacks; c != null; c = c.next) {
        c.run(frameTimeNanos);
    }
}

//CallbackRecord.java
public void run(long frameTimeNanos) {
    if (token == FRAME_CALLBACK_TOKEN) {
        ((FrameCallback)action).doFrame(frameTimeNanos);
    } else {
        ((Runnable)action).run();
    }
}
```

在 Choregrapher 中，除了 postCallback() 可以监听 VSync 信号然后执行 Runnable，还有可以通过 postFrameCallback() 监听 VSync 信号然后执行 FrameCallback。需要注意的是在最终调用到 postCallbackDelayedInternal() 时，参数 callbackType=Choreographer.CALLBACK_ANIMATION，参数token=FRAME_CALLBACK_TOKEN。

```
public void postFrameCallback(FrameCallback callback) {
    postFrameCallbackDelayed(callback, 0);
}

public void postFrameCallbackDelayed(FrameCallback callback, long delayMillis) {
    if (callback == null) {
        throw new IllegalArgumentException("callback must not be null");
    }
    postCallbackDelayedInternal(CALLBACK_ANIMATION,
            callback, FRAME_CALLBACK_TOKEN, delayMillis);
}
```

## View 测量布局绘制

### performTraversals

ViewRootImpl 的 performTraversals 方法中开始执行 View 体系的测量布局绘制流程：

```java
//ViewRootImpl.java
private void performTraversals() {
    final View host = mView;
    if (mFirst) {
        //回调view的onAttachToWindow()
        //将View所有的HandlerAction都已经交给 ViewRootImpl 去处理
    	host.dispatchAttachedToWindow(mAttachInfo, 0);
    }
    //执行ViewRootImpl.HandlerAction
    getRunQueue().executeActions(mAttachInfo.mHandler);
    ...
    // 执行 measure 逻辑（mStopped表示Activity是否处于stopped状态）
    boolean layoutRequested = mLayoutRequested && (!mStopped || mReportNextDraw);
    if (layoutRequested) {
        windowSizeMayChange |= measureHierarchy(host, lp, res, desiredWindowWidth, desiredWindowHeight);
    }
    ...
    // 执行 layout 逻辑（mStopped表示Activity是否处于stopped状态）
    final boolean didLayout = layoutRequested && (!mStopped || mReportNextDraw);
    if (didLayout) {
        performLayout(lp, mWidth, mHeight);
    }
    ...
    // 通知布局结束
    mAttachInfo.mTreeObserver.dispatchOnGlobalLayout();
    ...
    // 执行 draw 逻辑
    boolean cancelDraw = mAttachInfo.mTreeObserver.dispatchOnPreDraw() || !isViewVisible;
    if(!cancelDraw){
        performDraw();
    }
    ...
}
```

在 View.dispatchAttachedToWindow() 中会完成 View.mAttachInfo 的赋值及 View.onAttachToWindow() 的回调。

```java
//View.java
void dispatchAttachedToWindow(AttachInfo info, int visibility) {
    mAttachInfo = info;
    if (mRunQueue != null) {
        //所有的HandlerAction都已经交给 ViewRootImpl 去处理
    	mRunQueue.executeActions(info.mHandler);
    	mRunQueue = null;
	}
    onAttachedToWindow();
}

//View.java
public boolean post(Runnable action) {
    final AttachInfo attachInfo = mAttachInfo;
    //在View.dispatchAttachedToWindow时才会给mAttachInfo赋值
    if (attachInfo != null) {
        return attachInfo.mHandler.post(action);
    }
    //HandlerActionQueue内包含一个HandlerAction数组，用于存放action
    getRunQueue().post(action);
    return true;
}

```

注意到在 ViewRootImpl 和 View 中都有 getQueue().executeActions() 相关操作，这是基于 Handler 的延迟操作。根据 [Window](Window.md) 中提到的 View 的测量是发生在 onResume 中的，所以在 onCreate 中 View 可以通过 post(Runnable) 来获取宽高信息。

```java
public class HandlerActionQueue {
    private HandlerAction[] mActions;
    private int mCount;
    // 这个就是我们在外边调用的 post 方法，最终会调用到 postDelayed 方法
    public void post(Runnable action) {
        postDelayed(action, 0);
    }
    // 将传入的 Runnable 对象存入数组中，等待调用
    public void postDelayed(Runnable action, long delayMillis) {
        final HandlerAction handlerAction = new HandlerAction(action, delayMillis);

        synchronized (this) {
            if (mActions == null) {
                mActions = new HandlerAction[4];
            }
            mActions = GrowingArrayUtils.append(mActions, mCount, handlerAction);
            mCount++;
        }
    }
    // 这里才是真的执行方法
    public void executeActions(Handler handler) {
        synchronized (this) {
            final HandlerAction[] actions = mActions;
            for (int i = 0, count = mCount; i < count; i++) {
                final HandlerAction handlerAction = actions[i];
                handler.postDelayed(handlerAction.action, handlerAction.delay);
            }
            mActions = null;
            mCount = 0;
        }
    }
}

//ViewRootImpl.java
static HandlerActionQueue getRunQueue() {
    // sRunQueues 是 ThreadLocal<HandlerActionQueue> 对象
    HandlerActionQueue rq = sRunQueues.get();
    if (rq != null) {
        return rq;
    }
    rq = new HandlerActionQueue();
    sRunQueues.set(rq);
    return rq;
}
```

总结一下：

- View.post 方法会为当前 View 对象初始化一个 HandlerActionQueue ，并将 Runnable 入队存储；

- 等在 ViewRootImpl.performTraversals 中递归调用到 View.dispatchAttachedToWindow 时，会将 ViewRootImpl 的 Handler 对象传下来，然后通过这个 Handler 将最初的 Runnable 发送到 UI 线程（消息队列中）等待执行，并将 View 的 HandlerActionQueue 对象置空，方便回收；

- ViewRootImpl.performTraversals 继续执行，才会为 UI 线程首次初始化 HandlerActionQueue 对象，并通过 ThreadLocal 进行存储，然后执行 executeActions 将 Runnable 放入 Handler 队列中

### Measure 流程

根据 Window 的大小，xml 布局文件以及 View 的相关属性的设置，来计算出每个 View 的大小尺寸。完成后会确定的 View 的 mMeasuredWidth 和 mMeasuredHeight 属性。

#### MeasureSpec

MeasureSpec 是 View 测量过程中体现测量值的一个静态类，它代表一个 32 位 int 值，高 2 位代表 SpecMode，低 30 位代表 SpecSize。SpecMode 指的是测量模式，SpecSize 指的是某种测量模式下的大小。

```java
public static class MeasureSpec {
    private static final int MODE_SHIFT = 30;
    // 用于去高2位的模
    private static final int MODE_MASK  = 0x3 << MODE_SHIFT;
    // 三种测量模式
    public static final int UNSPECIFIED = 0 << MODE_SHIFT;
    public static final int EXACTLY     = 1 << MODE_SHIFT;
    public static final int AT_MOST     = 2 << MODE_SHIFT;
    
    // 将 SpecMode 和 SpecSize 组合
    public static int makeMeasureSpec(int size, int mode) {
        if (sUseBrokenMakeMeasureSpec) {
            return size + mode;
        } else {
            return (size & ~MODE_MASK) | (mode & MODE_MASK);
        }
    }
    
    // 解析测量模式
    public static int getMode(int measureSpec) {
        return (measureSpec & MODE_MASK);
    }
    
    // 解析测量大小
    public static int getSize(int measureSpec) {
        return (measureSpec & ~MODE_MASK);
    }
}
```

SpecMode 的三个值，代表着父 View 对子 View 测量时的要求：

- UNSPECIFIED：父 View 不对子 View 的大小做限制
- EXACTLY：父 View 已经计算出子 View 的确定的大小
- AT_MOST：父 View 最多允许子 View 的大小

#### ViewRootImpl.measureHierarchy()

ViewRootImpl.measureHierarchy 开启了测量流程，通过 getRootMeasureSpec() 获取 Window 的大小来设置顶层 MeasureSpec。然后将顶层 MeasureSpec 传入 View.measure 流程。

```java
private boolean measureHierarchy(final View host, final WindowManager.LayoutParams lp,
        final Resources res, final int desiredWindowWidth, final int desiredWindowHeight) {
    int childWidthMeasureSpec;
    int childHeightMeasureSpec;
    boolean windowSizeMayChange = false;
    boolean goodMeasure = false;
    if (lp.width == ViewGroup.LayoutParams.WRAP_CONTENT) {
        int baseSize = 0;
        if (mTmpValue.type == TypedValue.TYPE_DIMENSION) {
            baseSize = (int)mTmpValue.getDimension(packageMetrics);
        }
        
        if (baseSize != 0 && desiredWindowWidth > baseSize) {
            //getRootMeasureSpec/getChildMeasureSpec 是相同的逻辑
            //baseSize代表Window外部的限制宽高
            //lp是Window自身布局对宽高的要求
            childWidthMeasureSpec = getRootMeasureSpec(baseSize, lp.width);
            childHeightMeasureSpec = getRootMeasureSpec(desiredWindowHeight, lp.height);
            performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
            
            if ((host.getMeasuredWidthAndState()&View.MEASURED_STATE_TOO_SMALL) == 0) {
                goodMeasure = true;
            } else {
                baseSize = (baseSize+desiredWindowWidth)/2;
                childWidthMeasureSpec = getRootMeasureSpec(baseSize, lp.width);
                performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
                if ((host.getMeasuredWidthAndState()&View.MEASURED_STATE_TOO_SMALL) == 0) {
                    goodMeasure = true;
                }
            }
        }
    }
    if (!goodMeasure) {
        childWidthMeasureSpec = getRootMeasureSpec(desiredWindowWidth, lp.width);
        childHeightMeasureSpec = getRootMeasureSpec(desiredWindowHeight, lp.height);
        performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
        if (mWidth != host.getMeasuredWidth() || mHeight != host.getMeasuredHeight()) {
            windowSizeMayChange = true;
        }
    }
    return windowSizeMayChange;
}

private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
    if (mView == null) {
        return;
    }
    try {
        mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    } finally {
    }
}

private static int getRootMeasureSpec(int windowSize, int rootDimension) {
    int measureSpec;
    switch (rootDimension) {
    case ViewGroup.LayoutParams.MATCH_PARENT:
        measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.EXACTLY);
        break;
    case ViewGroup.LayoutParams.WRAP_CONTENT:
        measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.AT_MOST);
        break;
    default:
        measureSpec = MeasureSpec.makeMeasureSpec(rootDimension, MeasureSpec.EXACTLY);
        break;
    }
    return measureSpec;
}
```

#### View.meaure()

View.measure 流程中，会有以下几个关键的变量或标识赋值：

- forceLayout 变量根据 PFLAG_FORCE_LAYOUT 标识判断是否是强制重新 measure
- needsLayout 变量根据 MeasureSpec 提供的值与现有值比较判断是否重新 measure
- 在 View.onMeasure 后，mPrivateFlags 设置 PFLAG_LAYOUT_REQUIRED，用于 layout

```java
public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
    ...
    // 判断mPrivateFlags标记为强制重新布局（PFLAG_FORCE_LAYOUT）
    // 一般在View.requestLayout()里赋值
    final boolean forceLayout = (mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT;
    // 如果上次父布局给的测量结果与此次不同，那么表示尺寸变了
    final boolean specChanged = widthMeasureSpec != mOldWidthMeasureSpec || heightMeasureSpec != mOldHeightMeasureSpec;
    //是否是Exactly模式
    final boolean isSpecExactly = MeasureSpec.getMode(widthMeasureSpec) == MeasureSpec.EXACTLY && MeasureSpec.getMode(heightMeasureSpec) == MeasureSpec.EXACTLY;
    //当前测量的尺寸与父布局给的尺寸是否一致
    final boolean matchesSpecSize = getMeasuredWidth() ==  MeasureSpec.getSize(widthMeasureSpec) && getMeasuredHeight() ==  MeasureSpec.getSize(heightMeasureSpec);
    //是否需要重新布局，在6.0及其以下，sAlwaysRemeasureExactly=true
    final boolean needsLayout = specChanged && (sAlwaysRemeasureExactly || !isSpecExactly || !matchesSpecSize);
    //如果强制布局或者需要重新布局
    if (forceLayout || needsLayout) {
        ...
        onMeasure(widthMeasureSpec, heightMeasureSpec);
        ...
        // mPrivateFlags增加标记PFLAG_LAYOUT_REQUIRED，会调用onLayout(xx)
        mPrivateFlags |= PFLAG_LAYOUT_REQUIRED;
    }
    ...
}
```

View 的 measure 方法被 final 修饰，它会调用 View 的 onMeasure 方法，在 onMeasure 的最后要通过 setMeasuredDimension() 方法设置自身测量的值。

```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
            getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
}

// 如果没有背景返回默认的 0，否则取两者最大
protected int getSuggestedMinimumWidth() {
    return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
}

// 获取View的默认大小，除去MeasureSpec.UNSPECIFIED模式，返回的是父View对自己的尺寸要求
public static int getDefaultSize(int size, int measureSpec) {
    int result = size;
    int specMode = MeasureSpec.getMode(measureSpec);
    int specSize = MeasureSpec.getSize(measureSpec);
    switch (specMode) {
    case MeasureSpec.UNSPECIFIED:
        result = size;
        break;
    case MeasureSpec.AT_MOST:
    case MeasureSpec.EXACTLY:
        result = specSize;
        break;
    }
    return result;
}
```

#### ViewGroup.measure()

ViewGroup 没有重写 onMeasure()，只是提供了 measureChildren()/measureChildWithMargins() 来测量子 View。不同的 ViewGroup 根据自身特性，在处理了自身的 measure 逻辑后，还需要遍历所有子 View 的 measure() 来得到最后的尺寸大小。

measureChildren()/measureChildWithMargins() 来测量子 View。在测量过程中，getChildMeasureSpec() 有两个对宽高的限制要求：

- SpecMode 代表父 View 对测量的要求
- LayoutParams 则代表着开发者对 View 的测量要求
  - LayoutParams = -1 是 match_parent
  - LayoutParams = -2 是 wrap_content
  - LayoutParams >= 0 是具体的值

getChildMeasureSpec() 完成了 SpecMode 和 LayoutParams 两个限制方法的融合。

```java
protected void measureChildWithMargins(View child,
        int parentWidthMeasureSpec, int widthUsed,
        int parentHeightMeasureSpec, int heightUsed) {
    final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();
    
    // 通过开发者要求和父 View 布局要求一起计算出 measureSpec 值
    final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec, mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin + widthUsed, lp.width);
    final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec, mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin + heightUsed, lp.height);
    
    // 将前面计算出的 measureSpec 值传给子 View （对子View而言，这就是父View对它的布局要求）
    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}
/**
 * spec 是父布局要求
 * childDimension 是开发者要求
 * 有两种遍历方法，开发者的要求最大，最应该满足：
 * childDimension > 0 则 resultSize=childDimension resultMode=EXACTLY
 * 开发者要求match_parent specMode=EXACTLY 则 resultSize=size resultMode=EXACTLY
 * 开发者要求match_parent specMode=AT_MOST 则 resultSize=size resultMode=AT_MOST
 * 开发者要求wrap_content specMode=EXACTLY 则 resultSize=size resultMode=AT_MOST
 * 开发者要求wrap_content specMode=AT_MOST 则 resultSize=size resultMode=AT_MOST
 * 最后调用 MeasureSpec.makeMeasureSpec 合成 resultSize 和 resultMode
 */
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
     // specMode 是父布局的测量模式要求
    int specMode = MeasureSpec.getMode(spec);
    int specSize = MeasureSpec.getSize(spec);
    // size 是父布局要求 - padding 的值，specMode 是父布局的测量模式要求
    int size = Math.max(0, specSize - padding);
    int resultSize = 0;
    int resultMode = 0;
    switch (specMode) {
    case MeasureSpec.EXACTLY:
        if (childDimension >= 0) {
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            resultSize = size;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        }
        break;
    case MeasureSpec.AT_MOST:
        if (childDimension >= 0) {
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        }
        break;
    case MeasureSpec.UNSPECIFIED:
        if (childDimension >= 0) {
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
            resultMode = MeasureSpec.UNSPECIFIED;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
            resultMode = MeasureSpec.UNSPECIFIED;
        }
        break;
    }
    return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
}
```

一般来说，setMeasuredDimension() 的入参是一个开发者计算出来的值，但这个值很有可能不符合父 View 的要求，所以可以通过 resolveSizeAndState()，进行一次矫正。

```java
// size 是子View计算出来的大小
// measureSpec 是父View的大小要求限制
public static int resolveSizeAndState(int size, int measureSpec, int childMeasuredState) {
    final int specMode = MeasureSpec.getMode(measureSpec);
    final int specSize = MeasureSpec.getSize(measureSpec);
    final int result;
    switch (specMode) {
        case MeasureSpec.AT_MOST:
            if (specSize < size) {
                result = specSize | MEASURED_STATE_TOO_SMALL;
            } else {
                result = size;
            }
            break;
        case MeasureSpec.EXACTLY:
            result = specSize;
            break;
        case MeasureSpec.UNSPECIFIED:
        default:
            result = size;
    }
    return result | (childMeasuredState & MEASURED_STATE_MASK);
}
```

无论哪种 ViewGroup，其测量过程的核心思想没有变：

> 1、根据子布局的 layout_xx 参数，结合从父布局拿到测量结果，生成子布局的测量结果
>  2、将为子布局生成的测量结果传递给子布局，子布局进行1步骤。这是个递归的过程，当遇到的子布局是 View 时，递归结束，开始回溯
>  3、根据子布局自己测量后的结果，结合父布局给自己的测量结果，记录下自己的测量值，至此一个 ViewGroup 测量完毕

### Layout 流程

Layout 流程根据 Measure 中各个子 View 计算出的大小，来确定每个子 View 的位置。完成后会确定 View 的 mLeft、mRight、mTop、mBottom，也就确定了 getWidth() 和 getHeight() 的值

```java
//View.java
public final int getWidth() {
    return mRight - mLeft;
}

public final int getHeight() {
    return mBottom - mTop;
}
```

#### ViewRootImpl.performLayout()

在 ViewRootImpl 的 performTraversals 方法中调用 performLayout 开启了布局流程。其中调用 mView 的 layout 过程根据 measure 的结果，确定了 View 的摆放位置。

```java
//ViewRootImpl.java
private void performLayout(WindowManager.LayoutParams lp, int desiredWindowWidth, int desiredWindowHeight) {
    ...
    mInLayout = true;
    final View host = mView;
    if (host == null) {
        return;
    }
    host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());
    ...
    mInLayout = false;
}
```

#### View.layout()

View.measure 流程中，会有以下几个关键的变量或标识赋值：

- 判断 mPrivateFlags 设置 PFLAG_LAYOUT_REQUIRED 来开启 layout
- layout 完成后 mPrivateFlags 清除 PFLAG_LAYOUT_REQUIRED
- layout 完成后 mPrivateFlags 清除 PFLAG_FORCE_LAYOUT
- 在 setFrame() 中如果尺寸发生变化，mPrivateFlags 设置 PFLAG_DRAWN 标识并调用 invalidate

```java
//View.java
public void layout(int l, int t, int r, int b) {
    //记录当前的坐标值
    int oldL = mLeft;
    int oldT = mTop;
    int oldB = mBottom;
    int oldR = mRight;
    
    //新(父布局给的)的坐标值与当前坐标值不一致，则认为有改变
    boolean changed = isLayoutModeOptical(mParent) ?
            setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);
    
    //坐标改变或者是需要重新layout
    //PFLAG_LAYOUT_REQUIRED 是measure结束后设置的标记
    if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
        //调用onLayout方法，传入父布局传入的坐标
        onLayout(changed, l, t, r, b);
        //去除mPrivateFlags的PFLAG_LAYOUT_REQUIRED标记
        mPrivateFlags &= ~PFLAG_LAYOUT_REQUIRED;
        //监听的onLayoutChange回调，通过addOnLayoutChangeListener 设置
        ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnLayoutChangeListeners != null) {
            ArrayList<OnLayoutChangeListener> listenersCopy = (ArrayList<OnLayoutChangeListener>)li.mOnLayoutChangeListeners.clone();
            int numListeners = listenersCopy.size();
            for (int i = 0; i < numListeners; ++i) {
                listenersCopy.get(i).onLayoutChange(this, l, t, r, b, oldL, oldT, oldR, oldB);
            }
        }
    }
    //清空强制布局标记，该标记在measure时判断是否需要onMeasure;在requestLayout时判断是否完成布局
    mPrivateFlags &= ~PFLAG_FORCE_LAYOUT;
    mPrivateFlags3 |= PFLAG3_IS_LAID_OUT;
}

protected boolean setFrame(int left, int top, int right, int bottom) {
    boolean changed = false;
    //当前坐标值与新的坐标值不一致，则重新设置
    if (mLeft != left || mRight != right || mTop != top || mBottom != bottom) {
        changed = true;
        //记录PFLAG_DRAWN标记位
        int drawn = mPrivateFlags & PFLAG_DRAWN;
        ...        
        //最终执行Draw流程
        boolean sizeChanged = (newWidth != oldWidth) || (newHeight != oldHeight);
        invalidate(sizeChanged);
        
        //PFLAG_HAS_BOUNDS表示已经完成布局
        mPrivateFlags |= PFLAG_HAS_BOUNDS;
        
        //将新的坐标值记录
        mLeft = left;
        mTop = top;
        mRight = right;
        mBottom = bottom;
        //设置坐标值给RenderNode
        mRenderNode.setLeftTopRightBottom(mLeft, mTop, mRight, mBottom);
        
        if (sizeChanged) {
            //调用sizeChange，在该方法里，我们已经能够拿到View宽、高值
            sizeChange(newWidth, newHeight, oldWidth, oldHeight);
        }
        ...
    }
    return changed;
}
```

当自身的大小位置确定后，就可以调用 onLayout() 确定子View 的布局。

#### 获取 View 的尺寸位置方法

Activity 的启动过程和 View 的加载是异步的，在 onCreate、onStart、onResume 中均无法正确得到某个 View 的宽高信息，有以下方法可以解决：

- 重写View.onSizeChanged(xx)方法获取
- 注册View.addOnLayoutChangeListener(xx)，在onLayoutChange(xx)里获取
- 重写View.onLayout(xx)方法获取
- View.post 一个 Runnable 对象获取

获取 View 的位置信息的方法有：

|                   方法                    |                   作用                   |  参考对象  |
| :---------------------------------------: | :--------------------------------------: | :--------: |
| getLeft()/getTop()/getRight()/getBottom() |      获得 View 相对于父 View 的坐标      |  父 View   |
|           getLocationInWindow()           |    获得 View 相对于窗口 Window 的坐标    | 窗口Window |
|           getLocationOnScreen()           |        获得 View 相对于屏幕的坐标        |    屏幕    |
|          getGlobalVisibleRect()           |    获得 View 可见部分相对于屏幕的坐标    | 屏幕左上角 |
|           getLocalVisibleRect()           | 获得 View 可见部分相对于自身左上角的坐标 | 自身左上角 |

### Draw 流程

#### ViewRootImpl.performDraw()

```java
//ViewRootImpl.java
private void performDraw() {
    final boolean fullRedrawNeeded = mFullRedrawNeeded || mReportNextDraw;
    mIsDrawing = true;
    ...
    try {
        boolean canUseAsync = draw(fullRedrawNeeded);
        ...
    } fially {
        mIsDrawing = false;
    }
}

private boolean draw(boolean fullRedrawNeeded) {
    if (mAttachInfo.mThreadedRenderer != null && mAttachInfo.mThreadedRenderer.isEnabled()) {
        ...
        //硬件加速绘制
        mAttachInfo.mThreadedRenderer.draw(mView, mAttachInfo, this);
    } else {
        ...
        //软件绘制
        if (!drawSoftware(surface, mAttachInfo, xOffset, yOffset, scalingRequired, dirty, surfaceInsets)) {
            return false;
        }
    }
}

private boolean drawSoftware(Surface surface, AttachInfo attachInfo, int xoff, int yoff, boolean scalingRequired, Rect dirty, Rect surfaceInsets) {
    final Canvas canvas;
    try {
        canvas = mSurface.lockCanvas(dirty);
    }
    ...
    try {
        mView.draw(canvas);
    }
    ...
}
```

使用软件绘制的时候，绘制操作都是通过 CPU 计算并写入 Bitmap，最终 Bitmap 直接渲染到屏幕上，当某个 View 需要刷新的时候，计算刷新的脏区域，在相交的地方重新绘制，重走 draw 过程。软件绘制的 Canvas 对象通过 lockCanvas 生成后，会传递给所有的子布局，所有 View 树共享一个 Canvas 对象，Canvas类型为：CompatibleCanvas。

使用硬件绘制的时候，绘制操作先被记录到 RenderNode 中，当渲染的时候，将这些操作集合交给 GPU 计算处理，当某个 View 需要刷新的时候，只需要重新生成与之相关的操作指令集，需要重走Draw过程，大大减少了无效的绘制请求，节约了CPU时间，提升程序运行流畅度。硬件绘制的 Canvas 对象每个都是重新生成的，Canvas类型为：RecordingCanvas。

#### 硬件加速的开启和关闭

硬件加速分为4个层级来控制：

- Application：在 Application 标签下，添加字段控制硬件加速的开启和关闭

  ```xml
  <application
  	android:hardwareAccelerated="true/false">
  </application>
  ```

- Activity：在 Activity 标签下，添加字段控制硬件加速的开启和关闭

  ```xml
  <activity
  	android:hardwareAccelerated="true/false">
  </activity>
  ```

- Window：只能开启硬件加速

  ```java
  getWindow().setFlags(
  	WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED,
  	WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED);
  ```

- View：只能关闭硬件加速

  ```java
  View.setLayerType(View.LAYER_TYPE_SOFTWARE, null);
  ```

#### View.draw()

View.draw 流程中，会有以下几个关键的变量或标识赋值：

- mPrivateFlags 清除 PFLAG_DRAWN 标识

View 的绘制统一在 draw() 方法中调度，按照顺序：

1. drawBackground()：绘制 View 的背景
2. onDraw()：View 本体的绘制
3. dispatchDraw()：子 View 开始绘制
4. onDrawForeground()：绘制前景

```java
//View.java
public void draw(Canvas canvas) {
    ...
    //标记PFLAG_DRAWN表示已经完成绘制
    mPrivateFlags = (privateFlags & ~PFLAG_DIRTY_MASK) | PFLAG_DRAWN;
    int saveCount;
    if (!dirtyOpaque) {
        // 第一步绘制背景
        drawBackground(canvas);
    }
    
    final int viewFlags = mViewFlags;
    //检查横向、纵向是否设置了边缘渐变效果
    boolean horizontalEdges = (viewFlags & FADING_EDGE_HORIZONTAL) != 0;
    boolean verticalEdges = (viewFlags & FADING_EDGE_VERTICAL) != 0;
    
    if (!verticalEdges && !horizontalEdges) {
        // 第二步绘制自己
         onDraw(canvas);
        
        // 第三步绘制子View
        dispatchDraw(canvas);
        
        //绘制自动填充的高亮(默认不会绘制)
        drawAutofilledHighlight(canvas);

        // 第四步绘制前景
        onDrawForeground(canvas);
        
         //第七步，绘制默认高亮，在touch mode模式基本不生效
        drawDefaultFocusHighlight(canvas);
        
        return;
    }
    ...
}
```

#### ViewGroup.dispatchDraw()

当 ViewGroup 没有背景时会绕过 onDraw() 直接调用 dispatchDraw()，所以 ViewGroup 绘制往往是写在 dispatchDraw() 中。

ViewGroup 默认会开启一个标志位 willNotDraw，如果需要 ViewGroup 也走 onDraw 方法，需要调用 setWillNotDraw() 来设置关闭。

重写 ViewGroup 的 isChildrenDrawingOrderEnabled() 方法，可以修改子 View 的绘制顺序。

```java
protected void dispatchDraw(Canvas canvas) {
    final int childrenCount = mChildrenCount;
    final View[] children = mChildren;
    // 预排序列表，usingRenderNodeProperties默认是true
    final ArrayList<View> preorderedList = usingRenderNodeProperties ? null : buildOrderedChildList();
    // 是否自定义顺序
    final boolean customOrder = preorderedList == null && isChildrenDrawingOrderEnabled();
    // 遍历子View
    for (int i = 0; i < childrenCount; i++) {
        // 获取当前需要绘制的View序号
        final int childIndex = getAndVerifyPreorderedIndex(childrenCount, i, customOrder);
        // 根据序号获取View
        final View child = getAndVerifyPreorderedView(preorderedList, children, childIndex);
        if ((child.mViewFlags & VISIBILITY_MASK) == VISIBLE || child.getAnimation() != null) {
            // 绘制子view
            more |= drawChild(canvas, child, drawingTime);
        }
    }
}

//获取子View的序号Index
private int getAndVerifyPreorderedIndex(int childrenCount, int i, boolean customOrder) {
    final int childIndex;
    if (customOrder) {
        final int childIndex1 = getChildDrawingOrder(childrenCount, i);
        childIndex = childIndex1;
    } else {
        childIndex = i;
    }
    return childIndex;
}

protected int getChildDrawingOrder(int childCount, int drawingPosition) {
    return drawingPosition;
}

//根据子View的序号Index获取子View
private static View getAndVerifyPreorderedView(ArrayList<View> preorderedList, View[] children,
        int childIndex) {
    final View child;
    if (preorderedList != null) {
        child = preorderedList.get(childIndex);
    } else {
        child = children[childIndex];
    }
    return child;
}
```

#### ViewGroup.drawChild()

ViewGroup.drawChild() 需要主动调用，他会保证子 View 的大小不超过自身的大小，具体代码在：

```java
boolean draw(Canvas canvas, ViewGroup parent, long drawingTime) {
    //没有开启硬件加速
    if (!drawingWithRenderNode) {
        //parentFlags 为父布局的flag
        //若是父布局需要裁剪子布局，也就是说clipChildren==true
        //那么就需要对canvas进行裁剪
        if ((parentFlags & ViewGroup.FLAG_CLIP_CHILDREN) != 0 && cache == null) {
            //软件绘制offsetForScroll==true
            if (offsetForScroll) {
                //裁剪canvas与子布局大小一致
                //sx,sy 是scroll值，没设置scroll时sx,sy都为0
                canvas.clipRect(sx, sy, sx + getWidth(), sy + getHeight());
            }
    }
}
```



["一文读懂"系列：Android屏幕刷新机制 - 掘金 (juejin.cn)](https://juejin.cn/post/7163858831309537294)

[最全的View绘制流程（下）— Measure、Layout、Draw - 简书 (jianshu.com)](https://www.jianshu.com/p/3366e4bec7ce)

[Android：6种高效 & 准确获取View坐标位置的方式 - 掘金 (juejin.cn)](https://juejin.cn/post/6844903975175602189)

[Android 自定义View之Draw过程(下) - 简书 (jianshu.com)](https://www.jianshu.com/p/76b8bd023fee)

[遇到个难题，怎么修改子View绘制顺序？ (qq.com)](https://mp.weixin.qq.com/s?__biz=MzAxMTI4MTkwNQ==&mid=2650837638&idx=2&sn=df389d0d84dc4c847f9e58a1f9f7f70f&chksm=80b74618b7c0cf0ebb8c355b3f1f9bebac3f686013f0de42c900c1aa00dd789bf48a916bda4b)

[Android clipChildren 原来要这么用？ (qq.com)](https://mp.weixin.qq.com/s?__biz=MzAxMTI4MTkwNQ==&mid=2650837193&idx=2&sn=28f6afc3b1acda50ce0308cd37648990&chksm=80b74457b7c0cd413e25bc16fbe9456928238242da3d506d2fb8eee693a9210481405bb29aea)

