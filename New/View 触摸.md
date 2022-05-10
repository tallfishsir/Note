# View 触摸

## MotionEvent

MotionEvent 是手指触碰屏幕后产生的事件，主要包括单点触控和多点触控，多点触碰多了记录手指的功能。

MotionEvent 事件拥有位置信息

- getX/getY 获取触摸事件相对于当前 View 左上角的坐标
- getRawX/getRawY 获取触摸事件相对于屏幕左上角的坐标

### 触控事件类型

常见的触控事件类型分为：

- ACTION_DOWN：手指初次触碰到屏幕触发
- ACTION_UP：手指最后一个离开屏幕触发
- ACTION_MOVE：手指在屏幕上移动触发
- ACTION_CANCEL：事件先交给当前 View 后被上层拦截时触发
- ACTION_OUTSIDE：手指不在控件区域时触发
- ACTION_POINTER_DOWN：有非主要的手指按下(**即按下之前已经有手指在屏幕上**)
- ACTION_POINTER_UP：有非主要的手指抬起(**即抬起之后仍然有手指在屏幕上**)

通过 getAction/getActionMasked 可以获取触控事件类型，getAction() 方法返回一个 int 类型的值，共有 32 位（0x00000000），最低 8 位(0x000000**ff**) 表示事件类型，再往前的 8 位(0x0000**ff**00) 表示事件编号。所以 getActionMasked() 就是对 getAction() 的结果进行过滤，只取事件类型。

```java
public final int getAction() {
    return nativeGetAction(mNativePtr);
}

public static final int ACTION_MASK = 0x00000ff;
public final int getActionMasked() {
    return nativeGetAction(mNativePtr) & ACTION_MASK;
}
```



### 触控事件手指编号

通过 getActionIndex() 可以获取触控事件的手指编号，这个方法只在 ACTION_DOWN、ACTION_UP、（前面这两个获取到的编号肯定是 0）ACTION_POINTER_DOWN、ACTION_POINTER_UP 这几个事件中有效。因为在手指移动时，getAction() 返回值永远是 0x00000002，没有手指编号信息。

```java
public static final int ACTION_POINTER_INDEX_MASK  = 0x0000ff00;
public static final int ACTION_POINTER_INDEX_SHIFT = 8;
public final int getActionIndex() {
    return (nativeGetAction(mNativePtr) & ACTION_POINTER_INDEX_MASK) >> ACTION_POINTER_INDEX_SHIFT;
}
```

但是通过本方法获取到 PointIndex 是会变化的，Index 变化有以下几个特点：

1. 从 0 开始，自动正常。（ACTION_DOWN 时事件编码就为0）
2. 如果之前落下的手指抬起，后面的手指 Index 就会减小。
3. Index 的变化趋向于第一次落下的数值

### 触控事件手指id

因为 PointIndex 会变化，所以一般都是利用 getActionIndex() 获取编号后，再通过 getPointerId() 来获取 pointerId ，pointerId 是一个从手指按下到手指抬起都不会变换的值，用做手指的唯一值，不会受到其他手指抬起和落下的影响。

```java
public final int getPointerId(int pointerIndex) {
    return nativeGetPointerId(mNativePtr, pointerIndex);
}
```

### 手指编号和触控事件手指id转换

getPointerId() 可以从 pointIndex 查找到 pointId， findPointerIndex 从 pointId 查找到 pointIndex。

```java
public final int findPointerIndex(int pointerId) {
    return nativeFindPointerIndex(mNativePtr, pointerId);
}
```

## 触控事件分发流程

### 从屏幕到 App

触碰屏时，首先触发硬件驱动，驱动收到事件后，将相应事件写入输入设备节点，Linux 内核会将硬件产生的触摸事件包装为 Event 存到 /dev/input/event[x] 目录下。

系统启动时，在 SystemServer 进程会启动管理事件输入的 InputManagerService 其内部，会启动一个读线程，也就是 InputReader，它会从 /dev/input/ 目录拿到任务，并且分发给 InputDispatcher 线程，然后进行统一的事件分发调度。

在Activity启动时会调用 ViewRootImpl.setView()，在ViewRootImpl.setView()过程中，也会同时注册 InputChannel。现在系统进程已经拿到输入事件了，App 中的Window与InputManagerService之间的通信实际上使用的InputChannel，InputChannel是一个pipe，底层实际是通过socket进行通信。

```java
public final class ViewRootImpl {

  public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
      requestLayout();
      // ...
      // Set up the input pipeline.
      mSyntheticInputStage = new SyntheticInputStage();
      InputStage viewPostImeStage = new ViewPostImeInputStage(mSyntheticInputStage);
      InputStage nativePostImeStage = new NativePostImeInputStage(viewPostImeStage, "aq:native-post-ime:" + counterSuffix);
      InputStage earlyPostImeStage = new EarlyPostImeInputStage(nativePostImeStage);
      InputStage imeStage = new ImeInputStage(earlyPostImeStage, "aq:ime:" + counterSuffix);
      InputStage viewPreImeStage = new ViewPreImeInputStage(imeStage);
      InputStage nativePreImeStage = new NativePreImeInputStage(viewPreImeStage, "aq:native-pre-ime:" + counterSuffix);
      mFirstInputStage = nativePreImeStage;
      mFirstPostImeInputStage = earlyPostImeStage;
      // 创建InputChannel
      mInputChannel = new InputChannel();
      // 通过Binder在SystemServer进程中完成InputChannel的注册
      mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                            getHostVisibility(), mDisplay.getDisplayId(),
                            mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                            mAttachInfo.mOutsets, mInputChannel);
  }
}
```

ViewRootImpl里的InputChannel就指向了正确的InputChannel, 作为Client端，App 进程的主线程就会监听 socket 客户端，当收到消息（输入事件）后，回调 NativeInputEventReceiver.handleEvent() 方法，最终会走到InputEventReceiver.dispachInputEvent 方法。

```java
//InputEventReceiver.java
private void dispatchInputEvent(int seq, InputEvent event) {
    mSeqMap.put(event.getSequenceNumber(), seq);
    onInputEvent(event); 
}

//ViewRootImpl.java ::WindowInputEventReceiver
final class WindowInputEventReceiver extends InputEventReceiver {
    public void onInputEvent(InputEvent event) {
       enqueueInputEvent(event, this, 0, true); 
    }
}

//ViewRootImpl.java
void enqueueInputEvent(InputEvent event,
        InputEventReceiver receiver, int flags, boolean processImmediately) {
    adjustInputEventForCompatibility(event);
    QueuedInputEvent q = obtainQueuedInputEvent(event, receiver, flags);

    QueuedInputEvent last = mPendingInputEventTail;
    if (last == null) {
        mPendingInputEventHead = q;
        mPendingInputEventTail = q;
    } else {
        last.mNext = q;
        mPendingInputEventTail = q;
    }
    mPendingInputEventCount += 1;

    if (processImmediately) {
        doProcessInputEvents(); 
    } else {
        scheduleProcessInputEvents();
    }
}
```

在 doProcessInputEvents 中会对 QueuedInputEvent 链表遍历并传递触控事件，事件分发完成后会调用 finishInputEvent，告知SystemServer进程的InputDispatcher线程，最终将该事件移除，完成此次事件的分发消费。

```java
   void doProcessInputEvents() {
           ...
        // Deliver all pending input events in the queue.
        while (mPendingInputEventHead != null) {
            QueuedInputEvent q = mPendingInputEventHead;
            mPendingInputEventHead = q.mNext;
            deliverInputEvent(q);
        }
        ....
    }

    private void deliverInputEvent(QueuedInputEvent q) {
        InputStage stage;
        ....
        //stage赋值操作
        ....
        if (stage != null) {
            stage.deliver(q);
        } else {
            finishInputEvent(q);
        }
    }

    abstract class InputStage {
        private final InputStage mNext;

        public InputStage(InputStage next) {
            mNext = next;
        }

        public final void deliver(QueuedInputEvent q) {
            if ((q.mFlags & QueuedInputEvent.FLAG_FINISHED) != 0) {
                forward(q);
            } else if (shouldDropInputEvent(q)) {
                finish(q, false);
            } else {
                traceEvent(q, Trace.TRACE_TAG_VIEW);
                final int result;
                try {
                    result = onProcess(q);
                } finally {
                    Trace.traceEnd(Trace.TRACE_TAG_VIEW);
                }
                apply(q, result);
            }
        }
    }
```

QueuedInputEvent 链表是在 ViewRootImpl.setView 中组建的，ViewPostImeInputStage 就是 View 的触控事件

- SyntheticInputStage。综合处理事件阶段，比如处理导航面板、操作杆等事件。
- ViewPostImeInputStage。视图输入处理阶段，比如按键、手指触摸等运动事件，我们熟知的view事件分发就发生在这个阶段。
- NativePostImeInputStage。本地方法处理阶段，主要构建了可延迟的队列。
- EarlyPostImeInputStage。输入法早期处理阶段。
- ImeInputStage。输入法事件处理阶段，处理输入法字符。
- ViewPreImeInputStage。视图预处理输入法事件阶段，调用视图view的dispatchKeyEventPreIme方法。
- NativePreImeInputStage。本地方法预处理输入法事件阶段。

在 ViewPostImeInputStage 中会先将触控事件传给 mView（DecorView），然后在 DecorView 的 dispatchTouchEvent 方法中传递给 Activity。因为 Activity 只有 Window 的引用，再传给 Window，Window 只有 DecorView 引用，又再传给 DecorView，开始流转。

```java
final class ViewPostImeInputStage extends InputStage {
    @Override
    protected int onProcess(QueuedInputEvent q) {
        if (q.mEvent instanceof KeyEvent) {
            return processKeyEvent(q);
        } else {
            final int source = q.mEvent.getSource();
            if ((source & InputDevice.SOURCE_CLASS_POINTER) != 0) {
                return processPointerEvent(q);
            } 
        }
    }

private int processPointerEvent(QueuedInputEvent q) {
    final MotionEvent event = (MotionEvent)q.mEvent;
    boolean handled = mView.dispatchPointerEvent(event)
    return handled ? FINISH_HANDLED : FORWARD;
}

public final boolean dispatchPointerEvent(MotionEvent event) {
    if (event.isTouchEvent()) {
        return dispatchTouchEvent(event);
    } else {
        return dispatchGenericMotionEvent(event);
    }
}
    
//DecorView.java
@Override
public boolean dispatchTouchEvent(MotionEvent ev) {
    //cb其实就是对应的Activity/Dialog
    final Window.Callback cb = mWindow.getCallback();
    return cb != null && !mWindow.isDestroyed() && mFeatureId < 0
            ? cb.dispatchTouchEvent(ev) : super.dispatchTouchEvent(ev);
}


//Activity.java
public boolean dispatchTouchEvent(MotionEvent ev) {
    if (ev.getAction() == MotionEvent.ACTION_DOWN) {
        onUserInteraction();
    }
    if (getWindow().superDispatchTouchEvent(ev)) {
        return true;
    }
    return onTouchEvent(ev);
}

//PhoneWindow.java
@Override
public boolean superDispatchTouchEvent(MotionEvent event) {
    return mDecor.superDispatchTouchEvent(event);
}

//DecorView.java
public boolean superDispatchTouchEvent(MotionEvent event) {
    return super.dispatchTouchEvent(event);
} 
```

### ViewGroup 触控事件分发

ViewGroup的事件分发机制实际上是一个责任链模式，整个过程汇总涉及到三个重要方法：

1. dispatchTouchEvent()：用于事件的分发，如果事件传递给当前 View，那么此方法一定会被调用。返回结果表示是否消耗当前事件。
2. onInterceptTouchEvent()：仅 ViewGroup 有这个方法，在 dispatchTouchEvent() 中调用，用来判断当前 View 是否拦截当前事件。返回结果表示是否拦截当前事件。
3. onTouchEvent()：在 dispatchTouchEvent() 中调用，用来处理点击事件，返回结果表示是否消耗当前事件。

在 ViewGroup中这三个方法的关系类似下面的伪代码：

```java
public boolean dispatchTouchEvent(MotionEvent event){
    boolean result = false;
    // 如果没有拦截交给子View
    if(!onInterceptTouchEvent(event)){
        result = child.dispatchTouchEvent(event);
    }
    // 如果事件没有被消费,询问自身onTouchEvent
    if(!result){
        result = onTouchEvent(event);
    }
    return result;
}
```

> 1. 同一个事件序列是从手指从接触屏幕开始，手指离开屏幕结束，在这个过程中产生的一系列事件。
> 2. 某个 ViewGroup 决定拦截了某事件，那么同一个事件序列内的所有事件都会直接交给它处理，并且它的 onInterceptTouchEvent 不会再被调用。
> 3. 某个 View 开始处理事件时，如果 onTouchEvent 中不消耗 ACTION_DOWN 事件（返回 false），那么同一个事件序列的其他事件也不会再交给它处理；如果 onTouchEvent 中不消耗 ACTION_DOWN 以外的事件，那么这个点击事件就会消失，父 View 的 onTouchEvent 不会被调用。最终消失的点击事件会传递给 Activity 处理。
> 4. ViewGroup 的 onInterceptTouchEvent 默认不拦截任何事件，返回 false。
> 5. View 的 onTouchEvent 默认消耗任何事件，返回 true。

```java
public boolean dispatchTouchEvent(MotionEvent ev) {
    ...
    boolean handled = false;
    if (onFilterTouchEventForSecurity(ev)) {
        final int action = ev.getAction();
        final int actionMasked = action & MotionEvent.ACTION_MASK;
        // 如果是ACTION_DOWN事件就刷新重置状态
        if (actionMasked == MotionEvent.ACTION_DOWN) {
            cancelAndClearTouchTargets(ev);
            resetTouchState();
        }
        // 检查当前ViewGroup是否拦截触控事件：
        // 1.如果是ACTION_DOWN事件
        // 2.如果 mFirstTouchTarget 不为空（也就是子View处理了此事件序列）
        // 就检查onInterceptTouchEvent返回值
        final boolean intercepted;
        if (actionMasked == MotionEvent.ACTION_DOWN
                || mFirstTouchTarget != null) {
            // 第二种情况，如果子View调用了requestDisallowInterceptTouchEvent方法设置 FLAG_DISALLOW_INTERCEPT 为 true
            // 就不会再判断当前View的onInterceptTouchEvent的返回值了，而是直接设置为不拦截
            final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
            if (!disallowIntercept) {
                intercepted = onInterceptTouchEvent(ev);
                ev.setAction(action);
            } else {
                intercepted = false;
            }
        } else {
            // 上面两种情况之外（当前ViewGroup拦截了事件），后续的事件给自己处理
            intercepted = true;
        }
        
        ...
        // 检查是否是取消事件或者设置了取消标志
        final boolean canceled = resetCancelNextUpFlag(this)
                || actionMasked == MotionEvent.ACTION_CANCEL;
        // 触控事件的处理目标
        TouchTarget newTouchTarget = null;
        // 触控事件的处理目标是否发生了改变
        boolean alreadyDispatchedToNewTouchTarget = false;
        // 触控事件不是cancel，并且当前ViewGroup没有拦截事件，向子View传递
        if (!canceled && !intercepted) {
            // 如果是ACTION_DOWN事件，表明是触控事件系列开始，开始遍历查找子View是否消耗
            if (actionMasked == MotionEvent.ACTION_DOWN
                    || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                    || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                ...
                final int childrenCount = mChildrenCount;
                if (newTouchTarget == null && childrenCount != 0) {
                    ...
                    final View[] children = mChildren;
                    // 遍历所有子View，判断View是否能接收事件
                    for (int i = childrenCount - 1; i >= 0; i--) {
                        ...
                        // dispatchTransformedTouchEvent，第三个参数不为空，将触控事件传给子View
                        // 子View消耗了触控事件，返回true
                        if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)){
                            ...
                            //将消耗了触控事件的子View，设置为newTouchTarget和mFirstTouchTarget
                            newTouchTarget = addTouchTarget(child, idBitsToAssign);
                            alreadyDispatchedToNewTouchTarget = true;
                            break;
                        }
                    }
                }
                ...
            }
        }
        // mFirstTouchTarget == null 有两种情况
        // 1.当前ViewGroup在ACTION_DOWN时，就拦截了整个触控事件
        // 2.遍历子View，都没有处理触控事件
        if (mFirstTouchTarget == null) {
            // dispatchTransformedTouchEvent第三个参数为空，将触控事件传给自己
            handled = dispatchTransformedTouchEvent(ev, canceled, null,
                    TouchTarget.ALL_POINTER_IDS);
        } else {
            // mFirstTouchTarget有值，说明触控事件有View处理
            TouchTarget predecessor = null;
            TouchTarget target = mFirstTouchTarget;
            while (target != null) {
                final TouchTarget next = target.next;
                // alreadyDispatchedToNewTouchTarget为true&target == newTouchTarget，说明当前触控事件是MotionEvent.ACTION_DOWN
                // MotionEvent.ACTION_DOWN已经在上面的逻辑中处理了
                if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                    handled = true;
                } else {
                    // 其他类型的触控事件，先查看是否被拦截取消或者拦截
                    final boolean cancelChild = resetCancelNextUpFlag(target.child)
                            || intercepted;
                    // 如果cancelChild=true，dispatchTransformedTouchEvent会把MotionEvent.ACTION_CANCEL事件传给之前处理触控的子View
                    // 如果cancelChild=false，dispatchTransformedTouchEvent会把当前触控事件传给之前处理触控的子View
                    if (dispatchTransformedTouchEvent(ev, cancelChild,
                            target.child, target.pointerIdBits)) {
                        handled = true;
                    }
                    // 如果cancelChild=true，会把mFirstTouchTarget重新设置为null
                    if (cancelChild) {
                        if (predecessor == null) {
                            mFirstTouchTarget = next;
                        } else {
                            predecessor.next = next;
                        }
                        target.recycle();
                        target = next;
                        continue;
                    }
                }
                predecessor = target;
                target = next;
            }
        }
        ...
    }
    return handled;
}
```

在整个分发过程中，dispatchTransformedTouchEvent() 方法承担了多个重要的任务：

- 分发触控事件给子 View 处理
- 分发触控事件给自己处理
- 将原来的触控事件类型改为 ACTION_CANCEL 分发给子 View

```java
private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
        View child, int desiredPointerIdBits) {
    final boolean handled;
    // 如果参数cancel=true，就把触控事件类型改为ACTION_CANCEL
    final int oldAction = event.getAction();
    if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {
        event.setAction(MotionEvent.ACTION_CANCEL);
        // 如果参数child=null，就把触控事件传给自己处理
        if (child == null) {
            // super.dispatchTouchEvent 是View的dispatchTouchEvent
            handled = super.dispatchTouchEvent(event);
        } else {
            handled = child.dispatchTouchEvent(event);
        }
        event.setAction(oldAction);
        return handled;
    }
    ...
     // 如果参数child=null，就把触控事件传给自己处理
    if (child == null) {
        // super.dispatchTouchEvent 是View的dispatchTouchEvent
        handled = super.dispatchTouchEvent(transformedEvent);
    } else {
        // 将ViewGroup自身的ScrollX/ScrollY修正，然后传给子View处理
        final float offsetX = mScrollX - child.mLeft;
        final float offsetY = mScrollY - child.mTop;
        transformedEvent.offsetLocation(offsetX, offsetY);
        if (! child.hasIdentityMatrix()) {
            transformedEvent.transform(child.getInverseMatrix());
        }
        handled = child.dispatchTouchEvent(transformedEvent);
    }
    transformedEvent.recycle();
    return handled;
}
```

### View 触控事件分发

在 dispatchTransformedTouchEvent 方法中调用了 ViewGroup.super.dispatchTouchEvent(event)，就会将触控事件传给ViewGroup 自身处理，也就是 View.dispatchTouchEvent。

```java
public boolean dispatchTouchEvent(MotionEvent event) {
    ...
    boolean result = false;
    // 如果是 ACTION_DOWN 先停止滑动
    final int actionMasked = event.getActionMasked();
    if (actionMasked == MotionEvent.ACTION_DOWN) {
        stopNestedScroll();
    }
    if (onFilterTouchEventForSecurity(event)) {
        // 如果开发者设置了 OnTouchListener，并返回 ture 拦截了事件，就设置 result = true
        ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnTouchListener != null
                && (mViewFlags & ENABLED_MASK) == ENABLED
                && li.mOnTouchListener.onTouch(this, event)) {
            result = true;
        }
        // OnTouchListener没有设置或者返回 false 没有拦截，就调用当前View 的 onTouchEvent
        // 如果返回 ture 拦截了事件，就设置 result = true
        if (!result && onTouchEvent(event)) {
            result = true;
        }
    }
    if (actionMasked == MotionEvent.ACTION_UP ||
            actionMasked == MotionEvent.ACTION_CANCEL ||
            (actionMasked == MotionEvent.ACTION_DOWN && !result)) {
        stopNestedScroll();
    }
    return result;
}
```

View 的 onTouchListener 优先级高于 onTouchEvent，如果 OnTouchListener 没有设置或者返回 false 没有消耗事件，就会调用当前 View 的 onTouchEvent。

```java
public boolean onTouchEvent(MotionEvent event) {
    final float x = event.getX();
    final float y = event.getY();
    final int viewFlags = mViewFlags;
    final int action = event.getAction();
    //判断当前View是否可点击，不可点击的View不会消耗事件
    final boolean clickable = ((viewFlags & CLICKABLE) == CLICKABLE
            || (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)
            || (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE;
    //不可用状态还是会消耗点击事件，返回值依赖于是否可点击
    if ((viewFlags & ENABLED_MASK) == DISABLED) {
        if (action == MotionEvent.ACTION_UP && (mPrivateFlags & PFLAG_PRESSED) != 0) {
            setPressed(false);
        }
        mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
        return clickable;
    }
    //TouchDelegate是在不改变View大小的情况下，增加View的点击面积，内部点击逻辑和View的类似
    if (mTouchDelegate != null) {
        if (mTouchDelegate.onTouchEvent(event)) {
            return true;
        }
    }
    //如果当前View可点击，开始具体的处理
    if (clickable || (viewFlags & TOOLTIP) == TOOLTIP) {
        switch (action) {
            case MotionEvent.ACTION_UP:
                boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
                if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {
                    // 如果当前View长按的事件在postDelay中还没有触发
                    // mHasPerformedLongPress是false
                    if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {
                        // 移除长按监听
                        removeLongPressCallback();
                        if (!focusTaken) {
                            // PerformClick实现Runnable，内部执行performClick()方法
                            // performClick中如果OnClickListener不为空，就返回true拦截事件
                            if (mPerformClick == null) {
                                mPerformClick = new PerformClick();
                            }
                            if (!post(mPerformClick)) {
                                performClick();
                            }
                        }
                    }
                    if (mUnsetPressedState == null) {
                        mUnsetPressedState = new UnsetPressedState();
                    }
                    // UnsetPressedState 实现 Runnable，若干时间后 setPressed(false)
                    if (prepressed) {
                        postDelayed(mUnsetPressedState, ViewConfiguration.getPressedStateDuration());
                    } else if (!post(mUnsetPressedState)) {
                        mUnsetPressedState.run();
                    }
                    removeTapCallback();
                }
                mIgnoreNextUpEvent = false;
                break;
            case MotionEvent.ACTION_DOWN:
                if (event.getSource() == InputDevice.SOURCE_TOUCHSCREEN) {
                    mPrivateFlags3 |= PFLAG3_FINGER_DOWN;
                }
                mHasPerformedLongPress = false;
                boolean isInScrollingContainer = isInScrollingContainer();
                if (isInScrollingContainer) {
                    // 如果当前View在滚动的View中，CheckForTap继承Runnable，
                    // 用于若干时间后注册长按点击监听checkForLongClick
                    mPrivateFlags |= PFLAG_PREPRESSED;
                    if (mPendingCheckForTap == null) {
                        mPendingCheckForTap = new CheckForTap();
                    }
                    mPendingCheckForTap.x = event.getX();
                    mPendingCheckForTap.y = event.getY();
                    postDelayed(mPendingCheckForTap, ViewConfiguration.getTapTimeout());
                } else {
                    // 设置当前View为被按下状态
                    setPressed(true, x, y);
                    // 创建CheckForLongPress对象也是继承Runnable
                    // 在若干时间后 postDelay 出去执行performLongClick并设置 mHasPerformedLongPress = true;
                    checkForLongClick(0, x, y);
                }
                break;
            case MotionEvent.ACTION_CANCEL:
                if (clickable) {
                    setPressed(false);
                }
                removeTapCallback();
                removeLongPressCallback();
                mInContextButtonPress = false;
                mHasPerformedLongPress = false;
                mIgnoreNextUpEvent = false;
                mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
                break;
            case MotionEvent.ACTION_MOVE:
                if (clickable) {
                    drawableHotspotChanged(x, y);
                }
                if (!pointInView(x, y, mTouchSlop)) {
                    removeTapCallback();
                    removeLongPressCallback();
                    if ((mPrivateFlags & PFLAG_PRESSED) != 0) {
                        setPressed(false);
                    }
                    mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
                }
                break;
        }
        // 只要当前 View 可点击，就会返回 true，消耗事件
        return true;
    }
    return false;
}
```

## 滑动冲突

### 外部拦截法

点击事件都先经过 父View 的拦截处理，如果需要就拦截，如果不需要就不拦截。

1. 重写父 View 的 onInterceptTouchEvent() 方法，onInterceptTouchEvent() 中 ACTION_DOWN 返回 false，使得事件可以传递到子 View。如果一旦回 true，父 View 会直接拦截后续的事件，不会传递给子 View。
2. onInterceptTouchEvent() 中 ACTION_MOVE 根据需要，如果父 View 拦截就返回 true，如果子 View 就返回 false。
3. onInterceptTouchEvent() 中 ACTION_UP 返回 false。

```java
// 父view.java      
@Override
public boolean onInterceptTouchEvent(MotionEvent ev) {
    boolean intercepted = false;
    boolean parentCanIntercept;

    switch (ev.getActionMasked()) {
        case MotionEvent.ACTION_DOWN:
            intercepted = false;
            break;
        case MotionEvent.ACTION_MOVE:
            if (parentCanIntercept) {
                intercepted = true;
            } else {
                intercepted = false;
            }
            break;
        case MotionEvent.ACTION_UP:
            intercepted = false;
            break;
    }
    return intercepted;
}
```

### 内部拦截法

父 View 不拦截任何事件，所有的事件都先交给 子View ，如果子View 需要就直接消耗，否则就交给父View 处理。

1. 重写父 View 的 onInterceptTouchEvent() 方法，onInterceptTouchEvent() 的 ACTION_DOWN 返回 false 不拦截，剩下的都返回 true 拦截。
2. 重写子 View 的 dispatchTouchEvent() 方法，dispatchTouchEvent() 中 ACTION_DOWN 调用 parent.requestDisallowInterceptTouchEvent(true) 不允许父 View 拦截事件。
3. 子 View 的 dispatchTouchEvent() 中 ACTION_MOVE ，如果父 View 需要就调用  parent.requestDisallowInterceptTouchEvent(false) 允许父 View 拦截事件，而父 View 的 onInterceptTouchEvent 中除了 ACTION_DOWN 都拦截了事件。

```java
// 父view.java            
@Override
public boolean onInterceptTouchEvent(MotionEvent ev) {
    if (ev.getActionMasked() == MotionEvent.ACTION_DOWN) {
        return false;
    } else {
        return true;
    }
}

// 子view.java
@Override
public boolean dispatchTouchEvent(MotionEvent event) {
    boolean parentCanIntercept;
    
    switch (event.getActionMasked()) {
        case MotionEvent.ACTION_DOWN:
            getParent().requestDisallowInterceptTouchEvent(true);
            break;
        case MotionEvent.ACTION_MOVE:
            getParent().requestDisallowInterceptTouchEvent(!parentCanIntercept);
            break;
        case MotionEvent.ACTION_UP:
            break;
    }
    return super.dispatchTouchEvent(event);
}
```

## 手势检测

### 手势检测 GestureDetector

GestureDetector 一共有 5 种构造函数，但有 2 种被废弃了，1 种是重复的，其他两个：

- GestureDetector(Context context, GestureDetector.OnGestureListener listener)
- GestureDetector(Context context, GestureDetector.OnGestureListener listener, Handler handler)

第一个参数 Context，用于获取不同的 ViewConfiguration，由此来设置 touchSlop、doubleTapTouchSlop 等判断变量数值。

第三个参数 Handler，用于给 GestureDetector 提供一个 Looper，在通常情况下是不需这个 Handler 的，因为在主线程中创建 GestureDetector，那么它内部创建的 Handler 会自动获得主线程的 Looper，然而如果在一个没有创建 Looper 的子线程中创建 GestureDetector 则需要传递一个带有 Looper 的 Handler 给它，否则就会因为无法获取到 Looper 导致创建失败。

第二个参数 OnGestureListener，GestureDetector 内部有几个 Listener 接口，用来回调不同类型的触摸事件

- OnGestureListener：监听单击、滑动、长按等操作
  - onDown(MotionEvent e) 用户按下屏幕的时候的回调
  - onLongPress(MotionEvent e) 用户长按后触发，触发之后不会触发其他回调，直至松开
  - onScroll(MotionEvent e1, MotionEvent e2,float distanceX, float distanceY) 手指滑动的时候执行的回调
  - onFling(MotionEvent e1, MotionEvent e2, float velocityX,float velocityY) 用户执行抛操作之后的回调
  - onSingleTapUp(MotionEvent e) 用户手指松开（UP事件）的时候如果没有执行onScroll()和onLongPress()，就会回调这个，说明是一个点击抬起事件
- OnDoubleTapListener：监听双击和单击操作
  - onSingleTapConfirmed(MotionEvent e) 可以确认（通过单击DOWN后300ms没有下一个DOWN事件确认）这不是一个双击事件，而是一个单击事件的时候会回调
  - onDoubleTap(MotionEvent e) 可以确认这是一个双击事件的时候回调
  - onDoubleTapEvent(MotionEvent e) onDoubleTap 回调之后的输入事件（DOWN、MOVE、UP）都会回调这个方法（这个方法可以实现一些双击后的控制，如让View双击后变得可拖动等）

面所有的回调方法的返回值都是boolean类型，和View的事件传递机制一样，返回 true 表示消耗了事件，返回 flase 表示没有消耗。平常使用时可以直接继承 SimpleOnGestureListener ，因为 SimpleOnGestureListener 类实现了上面的接口，然后再重写我们所需要的回调方法。

在 GestureDetector 的 onTouchEvent 方法中处理触控事件

```java
public boolean onTouchEvent(MotionEvent ev) {
    //检查事件输入一致性，比如有事件只有up没有down
    if (mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onTouchEvent(ev, 0);
    }
    final int action = ev.getAction();
    //将当前触控事件克隆到mCurrentMotionEvent，方便后续判断
    mCurrentMotionEvent = MotionEvent.obtain(ev);
    mVelocityTracker.addMovement(ev);
    //检查多点触控时，是否是非主要手指抬起
    final boolean pointerUp =
            (action & MotionEvent.ACTION_MASK) == MotionEvent.ACTION_POINTER_U
    final int skipIndex = pointerUp ? ev.getActionIndex() : -1;
    final boolean isGeneratedGesture =
            (ev.getFlags() & MotionEvent.FLAG_IS_GENERATED_GESTURE) != 0;
    // 计算所有手指位置的中心焦点，并忽略非主要手指抬起时的位置
    float sumX = 0, sumY = 0;
    final int count = ev.getPointerCount();
    for (int i = 0; i < count; i++) {
        if (skipIndex == i) continue;
        sumX += ev.getX(i);
        sumY += ev.getY(i);
    }
    final int div = pointerUp ? count - 1 : count;
    final float focusX = sumX / div;
    final float focusY = sumY / div;
    
    boolean handled = false;
    switch (action & MotionEvent.ACTION_MASK) {
        case MotionEvent.ACTION_POINTER_DOWN:
            mDownFocusX = mLastFocusX = focusX;
            mDownFocusY = mLastFocusY = focusY;
            // 多点触控，非主要手指down时，取消长按监听
            cancelTaps();
            break;
        case MotionEvent.ACTION_POINTER_UP:
            mDownFocusX = mLastFocusX = focusX;
            mDownFocusY = mLastFocusY = focusY;
            // Check the dot product of current velocities.
            // If the pointer that left was opposing another velocity vector, 
            mVelocityTracker.computeCurrentVelocity(1000, mMaximumFlingVelocit
            final int upIndex = ev.getActionIndex();
            final int id1 = ev.getPointerId(upIndex);
            final float x1 = mVelocityTracker.getXVelocity(id1);
            final float y1 = mVelocityTracker.getYVelocity(id1);
            for (int i = 0; i < count; i++) {
                if (i == upIndex) continue;
                final int id2 = ev.getPointerId(i);
                final float x = x1 * mVelocityTracker.getXVelocity(id2);
                final float y = y1 * mVelocityTracker.getYVelocity(id2);
                final float dot = x + y;
                if (dot < 0) {
                    mVelocityTracker.clear();
                    break;
                }
            }
            break;
        case MotionEvent.ACTION_DOWN:
            if (mDoubleTapListener != null) {
                // DOUBLE_TAP_TIMEOUT内还有TAP事件，就是双击事件
                boolean hadTapMessage = mHandler.hasMessages(TAP);
                if (hadTapMessage) mHandler.removeMessages(TAP);
                //isConsideredDoubleTap包含识别为双击的特定条件
                if ((mCurrentDownEvent != null) && (mPreviousUpEvent != null)
                        && hadTapMessage
                        && isConsideredDoubleTap(mCurrentDownEvent, mPreviousUpEvent, ev)) {
                    mIsDoubleTapping = true;
                    // 回调双击监听
                    handled |= mDoubleTapListener.onDoubleTap(mCurrentDownEvent);
                    handled |= mDoubleTapListener.onDoubleTapEvent(ev);
                } else {
                    // 延迟发出单击事件，DOUBLE_TAP_TIMEOUT没有取消就是单击事件
                    mHandler.sendEmptyMessageDelayed(TAP, DOUBLE_TAP_TIMEOUT);
                }
            }
            mDownFocusX = mLastFocusX = focusX;
            mDownFocusY = mLastFocusY = focusY;
            if (mCurrentDownEvent != null) {
                mCurrentDownEvent.recycle();
            }
            mCurrentDownEvent = MotionEvent.obtain(ev);
            mAlwaysInTapRegion = true;
            mAlwaysInBiggerTapRegion = true;
            mStillDown = true;
            mInLongPress = false;
            mDeferConfirmSingleTap = false;
            mHasRecordedClassification = false;
            //处理长按事件
            if (mIsLongpressEnabled) {
                mHandler.removeMessages(LONG_PRESS);
                mHandler.sendMessageAtTime(
                        mHandler.obtainMessage(
                                LONG_PRESS,
                                TOUCH_GESTURE_CLASSIFIED__CLASSIFICATION__LONG
                                0 /* arg2 */),
                        mCurrentDownEvent.getDownTime()
                                + ViewConfiguration.getLongPressTimeout());
            }
            mHandler.sendEmptyMessageAtTime(SHOW_PRESS,
                    mCurrentDownEvent.getDownTime() + TAP_TIMEOUT);
            handled |= mListener.onDown(ev);
            break;
        case MotionEvent.ACTION_MOVE:
            if (mInLongPress || mInContextClick) {
                break;
            }
            final int motionClassification = ev.getClassification();
            final boolean hasPendingLongPress = mHandler.hasMessages(LONG_PRES
            final float scrollX = mLastFocusX - focusX;
            final float scrollY = mLastFocusY - focusY;
            if (mIsDoubleTapping) {
                // 如果前面down逻辑中判断是双击，后面的事件传给回调onDoubleTapEvent
                recordGestureClassification(
                        TOUCH_GESTURE_CLASSIFIED__CLASSIFICATION__DOUBLE_TAP);
                handled |= mDoubleTapListener.onDoubleTapEvent(ev);
            } else if (mAlwaysInTapRegion) {
                // down事件中会设置mAlwaysInTapRegion=true
                final int deltaX = (int) (focusX - mDownFocusX);
                final int deltaY = (int) (focusY - mDownFocusY);
                int distance = (deltaX * deltaX) + (deltaY * deltaY);
                int slopSquare = isGeneratedGesture ? 0 : mTouchSlopSquare;
                final boolean ambiguousGesture =
                        motionClassification == MotionEvent.CLASSIFICATION_AMB
                final boolean shouldInhibitDefaultAction =
                        hasPendingLongPress && ambiguousGesture;
                // 移动距离大于slopSquare，判断为sroll事件
                if (distance > slopSquare) {
                    recordGestureClassification(
                            TOUCH_GESTURE_CLASSIFIED__CLASSIFICATION__SCROLL);
                    handled = mListener.onScroll(mCurrentDownEvent, ev, scroll
                    mLastFocusX = focusX;
                    mLastFocusY = focusY;
                    mAlwaysInTapRegion = false;
                    mHandler.removeMessages(TAP);
                    mHandler.removeMessages(SHOW_PRESS);
                    mHandler.removeMessages(LONG_PRESS);
                }
                int doubleTapSlopSquare = isGeneratedGesture ? 0 : mDoubleTapT
                if (distance > doubleTapSlopSquare) {
                    mAlwaysInBiggerTapRegion = false;
                }
            } else if ((Math.abs(scrollX) >= 1) || (Math.abs(scrollY) >= 1)) {
                // 除了down事件后的scroll移动，判断为sroll事件
                recordGestureClassification(TOUCH_GESTURE_CLASSIFIED__CLASSIFI
                handled = mListener.onScroll(mCurrentDownEvent, ev, scrollX, s
                mLastFocusX = focusX;
                mLastFocusY = focusY;
            }
            final boolean deepPress =
                    motionClassification == MotionEvent.CLASSIFICATION_DEEP_PR
            if (deepPress && hasPendingLongPress) {
                mHandler.removeMessages(LONG_PRESS);
                mHandler.sendMessage(
                        mHandler.obtainMessage(
                              LONG_PRESS,
                              TOUCH_GESTURE_CLASSIFIED__CLASSIFICATION__DEEP_P
                              0 /* arg2 */));
            }
            break;
        case MotionEvent.ACTION_UP:
            mStillDown = false;
            MotionEvent currentUpEvent = MotionEvent.obtain(ev);
            if (mIsDoubleTapping) {
                // 如果前面down逻辑中判断是双击，回调onDoubleTapEvent
                recordGestureClassification(
                        TOUCH_GESTURE_CLASSIFIED__CLASSIFICATION__DOUBLE_TAP);
                handled |= mDoubleTapListener.onDoubleTapEvent(ev);
            } else if (mInLongPress) {
                //长按结束
                mHandler.removeMessages(TAP);
                mInLongPress = false;
            } else if (mAlwaysInTapRegion && !mIgnoreNextUpEvent) {
                //单击结束
                recordGestureClassification(
                        TOUCH_GESTURE_CLASSIFIED__CLASSIFICATION__SINGLE_TAP);
                handled = mListener.onSingleTapUp(ev);
                if (mDeferConfirmSingleTap && mDoubleTapListener != null) {
                    mDoubleTapListener.onSingleTapConfirmed(ev);
                }
            } else if (!mIgnoreNextUpEvent) {
                // 处理fling
                final VelocityTracker velocityTracker = mVelocityTracker;
                final int pointerId = ev.getPointerId(0);
                velocityTracker.computeCurrentVelocity(1000, mMaximumFlingVelo
                final float velocityY = velocityTracker.getYVelocity(pointerId
                final float velocityX = velocityTracker.getXVelocity(pointerId
                if ((Math.abs(velocityY) > mMinimumFlingVelocity)
                        || (Math.abs(velocityX) > mMinimumFlingVelocity)) {
                    handled = mListener.onFling(mCurrentDownEvent, ev, velocit
                }
            }
            if (mPreviousUpEvent != null) {
                mPreviousUpEvent.recycle();
            }
            // 设置mPreviousUpEvent
            mPreviousUpEvent = currentUpEvent;
            if (mVelocityTracker != null) {
                mVelocityTracker.recycle();
                mVelocityTracker = null;
            }
            mIsDoubleTapping = false;
            mDeferConfirmSingleTap = false;
            mIgnoreNextUpEvent = false;
            mHandler.removeMessages(SHOW_PRESS);
            mHandler.removeMessages(LONG_PRESS);
            break;
        case MotionEvent.ACTION_CANCEL:
            cancel();
            break;
    }
    if (!handled && mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onUnhandledEvent(ev, 0);
    }
    return handled;
}
```

### 缩放手势检测 ScaleGestureDetector

ScaleGestureDetector 的 Listener 有三个方法

- boolean onScaleBegin(ScaleGestureDetector detector) 缩放手势开始，当两个手指放在屏幕上的时候会调用该方法(只调用一次)，如果返回 false 则表示不使用当前这次缩放手势
- boolean onScaleBegin(ScaleGestureDetector detector) 缩放被触发(会调用0次或者多次)，如果返回 true 则表示当前缩放事件已经被处理，检测器会重新积累缩放因子，返回 false 则会继续积累缩放因子
- void onScaleEnd(ScaleGestureDetector detector) 缩放手势结束

ScaleGestureDetector 有三个重要方法

- getFocusX()：缩放中心 x 坐标
- getFocusY()：缩放中心 y 坐标
- getScaleFactor()：缩放因子



getScaleFactor()



[【带着问题学】Android事件分发8连问 (qq.com)](https://mp.weixin.qq.com/s?__biz=MzA5MzI3NjE2MA==&mid=2650262749&idx=1&sn=7a2726d9927f4866f1548e0dce1dcefc&chksm=88633db2bf14b4a4a34f4bc0a6b5d2687bb4330d52ab0805e06321f6c6690277bf675bf3e07d)

[Android 事件分发中你可能忽略掉的点！ (qq.com)](https://mp.weixin.qq.com/s?__biz=MzAxMTI4MTkwNQ==&mid=2650837456&idx=2&sn=d84c1609414788a490ba743e80f90d7d&chksm=80b7454eb7c0cc587a0da43520acb56521f5c3ac29d5aa95b956c1c52dacc2b7a9f42f79f466)

[Android事件分发机制详解：史上最全面、最易懂 - 简书 (jianshu.com)](https://www.jianshu.com/p/38015afcdb58)

[Android手势检测——GestureDetector全面分析_炎之铠的博客-CSDN博客_gesturedetector](https://blog.csdn.net/totond/article/details/77881180)







