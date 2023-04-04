# View 手势操作

## GestureDetector

GestureDetector 是一个用于检测触摸手势的工具，通过实现 onGestureListener、onDoubleTapListener、onContextClickListener，可以监听单击、双击、滑动、抛等操作。

### 核心类/接口

#### 构造函数

GestureDetector 构造函数中会初始化用于判断点击区域的 Slop，判断触发长按的时间等，构造函数参数：

- Context：用于 ViewConfiguration 获取 touchSlop、doubleTapTouchSlop 等变量数值
- OnGestureListener ：用于设置需要监听的事件
- Handler：用于监听过程中一些延时处理

```java
private final OnGestureListener mListener;
private OnDoubleTapListener mDoubleTapListener;
private OnContextClickListener mContextClickListener;

public GestureDetector(@Nullable @UiContext Context context,
        @NonNull OnGestureListener listener, @Nullable Handler handler) {
    if (handler != null) {
        mHandler = new GestureHandler(handler);
    } else {
        mHandler = new GestureHandler();
    }
    mListener = listener;
    if (listener instanceof OnDoubleTapListener) {
        setOnDoubleTapListener((OnDoubleTapListener) listener);
    }
    if (listener instanceof OnContextClickListener) {
        setContextClickListener((OnContextClickListener) listener);
    }
    init(context);
}

private void init(@UiContext Context context) {
    if (mListener == null) {
        throw new NullPointerException("OnGestureListener must not be null");
    }
    mIsLongpressEnabled = true;
    // Fallback to support pre-donuts releases
    int touchSlop, doubleTapSlop, doubleTapTouchSlop;
    if (context == null) {
        //noinspection deprecation
        touchSlop = ViewConfiguration.getTouchSlop();
        doubleTapTouchSlop = touchSlop; // Hack rather than adding a hidden method for this
        doubleTapSlop = ViewConfiguration.getDoubleTapSlop();
        //noinspection deprecation
        mMinimumFlingVelocity = ViewConfiguration.getMinimumFlingVelocity();
        mMaximumFlingVelocity = ViewConfiguration.getMaximumFlingVelocity();
        mAmbiguousGestureMultiplier = ViewConfiguration.getAmbiguousGestureMultiplier();
    } else {
        StrictMode.assertConfigurationContext(context, "GestureDetector#init");
        final ViewConfiguration configuration = ViewConfiguration.get(context);
        touchSlop = configuration.getScaledTouchSlop();
        doubleTapTouchSlop = configuration.getScaledDoubleTapTouchSlop();
        doubleTapSlop = configuration.getScaledDoubleTapSlop();
        mMinimumFlingVelocity = configuration.getScaledMinimumFlingVelocity();
        mMaximumFlingVelocity = configuration.getScaledMaximumFlingVelocity();
        mAmbiguousGestureMultiplier = configuration.getScaledAmbiguousGestureMultiplier();
    }
    mTouchSlopSquare = touchSlop * touchSlop;
    mDoubleTapTouchSlopSquare = doubleTapTouchSlop * doubleTapTouchSlop;
    mDoubleTapSlopSquare = doubleTapSlop * doubleTapSlop;
}
```

#### 监听接口

OnGestureListener 用于监听：

- boolean onDown(MotionEvent e)：用户按下屏幕的时候的回调
- boolean onLongPress(MotionEvent e)： 用户长按后触发，触发之后不会触发其他回调，直至松开
- boolean onScroll(MotionEvent e1, MotionEvent e2,float distanceX, float distanceY)： 手指滑动的时候执行的回调，e1 是开始的 DOWN 事件，e2 是前一个 MOVE 事件，distance 是当前 MOVE 事件与 e2 的位移量
- boolean onFling(MotionEvent e1, MotionEvent e2, float velocityX,float velocityY) 用户执行抛操作之后的回调，e1 是开始的 DOWN 事件，e2 是 UP 事件，velocity 是 UP 事件时的速度
- boolean onSingleTapUp(MotionEvent e) 用户手指松开（UP事件）的时候如果没有执行onScroll()和onLongPress()，就会回调这个，说明是一个点击抬起事件，但是不能区分是否双击事件的抬起

OnDoubleTapListener 用于监听：

- boolean onSingleTapConfirmed(MotionEvent e) 可以确认（通过单击DOWN后300ms没有下一个DOWN事件确认）这不是一个双击事件，而是一个单击事件的时候会回调
- boolean onDoubleTap(MotionEvent e) 可以确认这是一个双击事件的时候回调
- boolean onDoubleTapEvent(MotionEvent e) onDoubleTap 回调之后的输入事件（DOWN、MOVE、UP）都会回调这个方法（实现一些双击后的控制）

上面所有的回调方法的返回值都是 boolean 类型，和 View 的事件传递机制一样，返回 true 表示消耗了事件，返回 flase 表示没有消耗。所以需要注意的是：确保覆盖 onDown 方法并返回 true，否则可能导致其他事件无法触发。

### 手势监听

一般情况下，会创建 GestureDetector 实例后，会在 View.onTouchEvent() 中直接调用 GestureDetector .onTouchEvent() 并返回它的返回值。

```java
public boolean onTouchEvent(MotionEvent ev) {
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

## ScaleGestureDetector

ScaleGestureDetector 是一个用于监听手势缩放的工具，通过实现 OnScaleGestureListener 接口，可以监听缩放手势。

### 核心类/接口

#### 构造函数

ScaleGestureDetector 构造函数中会初始化 Slop 等变量

```java
public ScaleGestureDetector(@NonNull Context context, @NonNull OnScaleGestureListener listener,
                            @Nullable Handler handler) {
    mContext = context;
    mListener = listener;
    final ViewConfiguration viewConfiguration = ViewConfiguration.get(context);
    mSpanSlop = viewConfiguration.getScaledTouchSlop() * 2;
    mMinSpan = viewConfiguration.getScaledMinimumScalingSpan();
    mHandler = handler;
    // Quick scale is enabled by default after JB_MR2
    final int targetSdkVersion = context.getApplicationInfo().targetSdkVersion;
    if (targetSdkVersion > Build.VERSION_CODES.JELLY_BEAN_MR2) {
        setQuickScaleEnabled(true);
    }
    // Stylus scale is enabled by default after LOLLIPOP_MR1
    if (targetSdkVersion > Build.VERSION_CODES.LOLLIPOP_MR1) {
        setStylusScaleEnabled(true);
    }
}

public void setQuickScaleEnabled(boolean scales) {
    mQuickScaleEnabled = scales;
    if (mQuickScaleEnabled && mGestureDetector == null) {
        GestureDetector.SimpleOnGestureListener gestureListener =
                new GestureDetector.SimpleOnGestureListener() {
                    @Override
                    public boolean onDoubleTap(MotionEvent e) {
                        // Double tap: start watching for a swipe
                        mAnchoredScaleStartX = e.getX();
                        mAnchoredScaleStartY = e.getY();
                        mAnchoredScaleMode = ANCHORED_SCALE_MODE_DOUBLE_TAP;
                        return true;
                    }
                };
        mGestureDetector = new GestureDetector(mContext, gestureListener, mHandler);
    }
}

public void setStylusScaleEnabled(boolean scales) {
    mStylusScaleEnabled = scales;
}
```

#### 监听接口

OnScaleGestureListener 用于监听：

- boolean onScaleBegin(ScaleGestureDetector detector) 缩放手势开始，当两个手指放在屏幕上的时候会调用该方法(只调用一次)，如果返回 false 则表示不使用当前这次缩放手势
- boolean onScale(ScaleGestureDetector detector) 缩放被触发(会调用0次或者多次)，如果返回 true 则表示当前缩放事件已经被处理，检测器会重新积累缩放因子，返回 false 则会继续积累缩放因子。
- void onScaleEnd(ScaleGestureDetector detector) 缩放手势结束

### 手势监听

一般情况下，会创建 ScaleGestureDetector 实例后，会在 View.onTouchEvent() 中直接调用 GestureDetector .onTouchEvent() 并返回它的返回值。

```java
public boolean onTouchEvent(@NonNull MotionEvent event) {
    //记录event时间和action
    mCurrTime = event.getEventTime();
    final int action = event.getActionMasked();
    //mGestureDetector重写了onDoubleTap
    if (mQuickScaleEnabled) {
        mGestureDetector.onTouchEvent(event);
    }
    final int count = event.getPointerCount();
    //DOWN事件，如果还有scale操作，就结束并调用onScaleEnd
    if (action == MotionEvent.ACTION_DOWN || streamComplete) {
        if (mInProgress) {
            mListener.onScaleEnd(this);
            mInProgress = false;
            mInitialSpan = 0;
            mAnchoredScaleMode = ANCHORED_SCALE_MODE_NONE;
        } else if (inAnchoredScaleMode() && streamComplete) {
            mInProgress = false;
            mInitialSpan = 0;
            mAnchoredScaleMode = ANCHORED_SCALE_MODE_NONE;
        }
        if (streamComplete) {
            return true;
        }
    }
    final boolean configChanged = action == MotionEvent.ACTION_DOWN ||
            action == MotionEvent.ACTION_POINTER_UP ||
            action == MotionEvent.ACTION_POINTER_DOWN || anchoredScaleCancelled;
    final boolean pointerUp = action == MotionEvent.ACTION_POINTER_UP;
    final int skipIndex = pointerUp ? event.getActionIndex() : -1;
    // Determine focal point
    float sumX = 0, sumY = 0;
    final int div = pointerUp ? count - 1 : count;
    final float focusX;
    final float focusY;
    if (inAnchoredScaleMode()) {
        ...
    } else {
        //计算出多指情况下的中心点
        for (int i = 0; i < count; i++) {
            if (skipIndex == i) continue;
            sumX += event.getX(i);
            sumY += event.getY(i);
        }
        focusX = sumX / div;
        focusY = sumY / div;
    }
    // 计算到焦点的平均距离
    float devSumX = 0, devSumY = 0;
    for (int i = 0; i < count; i++) {
        if (skipIndex == i) continue;
        devSumX += Math.abs(event.getX(i) - focusX);
        devSumY += Math.abs(event.getY(i) - focusY);
    }
    final float devX = devSumX / div;
    final float devY = devSumY / div;
    final float spanX = devX * 2;
    final float spanY = devY * 2;
    final float span;
    if (inAnchoredScaleMode()) {
        span = spanY;
    } else {
        span = (float) Math.hypot(spanX, spanY);
    }
    //mSpanSlop 和 mMinSpan 都是从系统里面取得的预定义数值，该数值实际上影响的是缩放的灵敏度。
    final int minSpan = inAnchoredScaleMode() ? mSpanSlop : mMinSpan;
    if (!mInProgress && span >=  minSpan &&
            (wasInProgress || Math.abs(span - mInitialSpan) > mSpanSlop)) {
        mPrevSpanX = mCurrSpanX = spanX;
        mPrevSpanY = mCurrSpanY = spanY;
        mPrevSpan = mCurrSpan = span;
        mPrevTime = mCurrTime;
        //当用户移动的距离超过一定数值(数值大小由系统定义)后，会触发 onScaleBegin 方法
        mInProgress = mListener.onScaleBegin(this);
    }
    // Handle motion; focal point and span/scale factor are changing.
    if (action == MotionEvent.ACTION_MOVE) {
        mCurrSpanX = spanX;
        mCurrSpanY = spanY;
        mCurrSpan = span;
        boolean updatePrev = true;
        if (mInProgress) {
            //onScale返回值决定了是否重新计算缩放因子
            updatePrev = mListener.onScale(this);
        }
        if (updatePrev) {
            //重新计算缩放因子
            mPrevSpanX = mCurrSpanX;
            mPrevSpanY = mCurrSpanY;
            mPrevSpan = mCurrSpan;
            mPrevTime = mCurrTime;
        }
    }
    return true;
}
```

## 滑动机制

### View 移动实现

View 的移动除了可以通过改变 translation 属性，还有两种方法可以进行改变

#### layout 

在 View 进行绘制的时候，会调用 layout() 来设置显示的位置，因此我们可以通过修改 View 的 mLeft、mTop、mRight、mBottom 四个坐标属性来控制 View 的坐标。

```
@Override
public boolean onTouchEvent(MotionEvent ev) {
    int x = (int) ev.getX();
    int y = (int) ev.getY();
    switch (ev.getAction()){
        case MotionEvent.ACTION_DOWN:
            mLastX = x;
            mLastY = y;
            break;
        case MotionEvent.ACTION_MOVE:
            int offsetX = x - mLastX;
            int offsetY = y - mLastY;
            // 调整layout的四个坐标
            layout(getLeft() + offsetX, getTop() + offsetY, getRight() + offsetX, getBottom() + offsetY);
            // offsetLeftAndRight(offsetX);
            // offsetTopAndBottom(offsetY);
            break;
    }
    return true;
}
```

除此之外，Android 还提供了两个方法，用于快捷修改 layout 坐标，他们的底层还是通过 layout() 实现的。

```java
public void offsetLeftAndRight(int offset) {
    if (offset != 0) {
        ...
        mLeft += offset;
        mRight += offset;
        mRenderNode.offsetLeftAndRight(offset);
        if (isHardwareAccelerated()) {
            invalidateViewProperty(false, false);
            invalidateParentIfNeededAndWasQuickRejected();
        } else {
            if (!matrixIsIdentity) {
                invalidateViewProperty(false, true);
            }
            invalidateParentIfNeeded();
        }
        notifySubtreeAccessibilityStateChangedIfNeeded();
    }
}

public void offsetTopAndBottom(int offset) {
    if (offset != 0) {
        ...
        mTop += offset;
        mBottom += offset;
        mRenderNode.offsetTopAndBottom(offset);
        if (isHardwareAccelerated()) {
            invalidateViewProperty(false, false);
            invalidateParentIfNeededAndWasQuickRejected();
        } else {
            if (!matrixIsIdentity) {
                invalidateViewProperty(false, true);
            }
            invalidateParentIfNeeded();
        }
        notifySubtreeAccessibilityStateChangedIfNeeded();
    }
}
```

#### Scroll

View 内部还有两个属性 mScrollX、mScrollY ，与 translation 不同的是，它描述的是 View 的内容的滑动距离。在 View 的坐标系下，从计算关系来看：

- mScrollX  = View 左边缘坐标 - View 内容的左边缘坐标
- mScrollY  = View 上边缘坐标 - View 内容的上边缘坐标

本质上，在 ViewGroup.draw 过程中，scroll 属性还是通过 canvas.translation() 来完成偏移：

```java
protected void dispatchDraw(Canvas canvas) {
	...
	if ((child.mViewFlags & VISIBILITY_MASK) == VISIBLE || child.getAnimation() != null) {
		more |= drawChild(canvas, child, drawingTime);
    }
    ...
}

protected boolean drawChild(Canvas canvas, View child, long drawingTime) {
        return child.draw(canvas, this, drawingTime);
}

boolean draw(Canvas canvas, ViewGroup parent, long drawingTime) {
    ...
	int sx = 0;
	int sy = 0;
	if (!drawingWithRenderNode) {
		//空实现，由开发实现，用于更新mScollX和mScrollY，一般配合Scroller使用
	    computeScroll();
	    sx = mScrollX;
	    sy = mScrollY;
	}
	...
	if (offsetForScroll) {
        //scroll数值最终在translate()中体现
		canvas.translate(mLeft - sx, mTop - sy);
	} 
	...
	if ((parentFlags & ViewGroup.FLAG_CLIP_CHILDREN) != 0 && cache == null) {
        if (offsetForScroll) {
            //scroll数值在clipRect中被修正
            canvas.clipRect(sx, sy, sx + getWidth(), sy + getHeight());
        } else {
            if (!scalingRequired || cache == null) {
                canvas.clipRect(0, 0, getWidth(), getHeight());
            } else {
                canvas.clipRect(0, 0, cache.getWidth(), cache.getHeight());
            }
        }
    }
}
```

根据 mLeft - sx, mTop - sy 公式可以推导出：在 View 的坐标系下， mScrollX 和 mScrollY 与 View 的移动方向是相反的。View 内容向左/上移动，mScroll 取值是正数，View 内容向右/下移动，mScroll 取值是负数。

View 提供了两种方法改变 Scroll 属性值：

- scrollTo()：相对于父 View 的绝对滑动
- scrollBy()：相对于当前位置的滑动

```java
public void scrollTo(int x, int y) {
    if (mScrollX != x || mScrollY != y) {
        int oldX = mScrollX;
        int oldY = mScrollY;
        mScrollX = x;
        mScrollY = y;
        invalidateParentCaches();
        onScrollChanged(mScrollX, mScrollY, oldX, oldY);
        if (!awakenScrollBars()) {
            postInvalidateOnAnimation();
        }
    }
}

public void scrollBy(int x, int y) {
    scrollTo(mScrollX + x, mScrollY + y);
}
```

### Scroller

由于 scrollTo() 完成的滑动都是瞬时性的，Scroller 是实现滑动动画的一种方式，核心思想是将滑动的距离分成若干个小滑动并在一段时间内完成。

#### 使用方式

使用 Scroller 时需要完成步骤：

- 创建 Scroller 实例
- 调用 Scroller.startScroll() 传入开始和结束数值，并调用 invalidate() 开启重绘
- 重写 View.computeScroll() 方法，调用 Scroller.computeScrollOffset() 计算新的 Scroll 数值，然后调用 scrollTo() 传入计算后的 Scroll 数值，并调用 postInvalidate() 开启下一次重绘

```java
Scroller mScroller;

public TestView(Context context, AttributeSet attrs, int defStyleAttr) {
	super(context, attrs, defStyleAttr);
	mScroller = new Scroller(context);
}

//滑动一段距离
public void startScrollBy(int dx,int dy) {
	//开始滑动前，强制停止上一次滑动
	mScroller.forceFinished(true);
	//获取当前滑动数值
	int startX = getScrollX();
	int startY = getScrollY();
	//传入开始和结束数值
	mScroller.startScroll(startX,startY, dx, dy,1000);
	//调用 invalidate() 开启重绘
	invalidate();
}

public void startFling(int dx,int dy) {
	//开始滑动前，强制停止上一次滑动
	mScroller.forceFinished(true);
	//获取当前滑动数值
	int startX = getScrollX();
	int startY = getScrollY();
	//传入开始和结束数值
	mScroller.fling(startX,startY, -xVelocity,-yVelocity,-1000,1000,-1000,2000);
	//调用 invalidate() 开启重绘
	invalidate();
}

@Override
public void computeScroll() {
    super.computeScroll();
    //计算新的 Scroll 数值
    if (mScroller.computeScrollOffset()) {
        //实际滑动
        scrollTo(mScroller.getCurrX(),mScroller.getCurrY());
        // 开启下一次重绘
        if (mScroller.getCurrX() == getScrollX()
                && mScroller.getCurrY() == getScrollY() ) {
            postInvalidate();
        }
    } else {
        Log.d(TAG, "computeScroll is over");
    }
}
```

#### 原理分析

Scroller.startScroll() 的作用只是记录滑动的开始结束数值，正常的计算过程是在 computeScrollOffset() 中实现。通过调用 invalidate() 重绘，在 ViewGroup.draw() 中执行 View.computeScroll() 计算出 mCurrentScroll 数值，再调用 ScrollTo() 完成真正的滑动，直到达到滑动目标。

```java
public class Scroller  {
	//滑动开始的坐标
	private int mStartX;
	private int mStartY;
	
	//滑动结束的坐标
	private int mFinalX;
	private int mFinalY;
	
	//computeScrollOffset()后计算的坐标
	private int mCurrX;
	private int mCurrY;
	
	//computeScrollOffset()中计算的插值器
	private final Interpolator mInterpolator;
	
	//startScroll()开始时间
	private long mStartTime;
	
	public void startScroll(int startX, int startY, int dx, int dy, int duration) {
		mMode = SCROLL_MODE;
		mFinished = false;
		mDuration = duration;
		mStartTime = AnimationUtils.currentAnimationTimeMillis();
		mStartX = startX;
		mStartY = startY;
		mFinalX = startX + dx;
		mFinalY = startY + dy;
		mDeltaX = dx;
		mDeltaY = dy;
		mDurationReciprocal = 1.0f / (float) mDuration;
	}
	
	public boolean computeScrollOffset() {
		if (mFinished) {
			return false;
		}
		int timePassed = (int)(AnimationUtils.currentAnimationTimeMillis() - mStartTime);
		if (timePassed < mDuration) {
			switch (mMode) {
			case SCROLL_MODE:
				final float x = mInterpolator.getInterpolation(timePassed * mDurationReciprocal);
				mCurrX = mStartX + Math.round(x * mDeltaX);
				mCurrY = mStartY + Math.round(x * mDeltaY);
				break;
			case FLING_MODE:
				final float t = (float) timePassed / mDuration;
				final int index = (int) (NB_SAMPLES * t);
				float distanceCoef = 1.f;
				float velocityCoef = 0.f;
				if (index < NB_SAMPLES) {
					final float t_inf = (float) index / NB_SAMPLES;
					final float t_sup = (float) (index + 1) / NB_SAMPLES;
					final float d_inf = SPLINE_POSITION[index];
					final float d_sup = SPLINE_POSITION[index + 1];
					velocityCoef = (d_sup - d_inf) / (t_sup - t_inf);
					distanceCoef = d_inf + (t - t_inf) * velocityCoef;
				}
				mCurrVelocity = velocityCoef * mDistance / mDuration * 1000.0f;
				
				mCurrX = mStartX + Math.round(distanceCoef * (mFinalX - mStartX));
				// Pin to mMinX <= mCurrX <= mMaxX
				mCurrX = Math.min(mCurrX, mMaxX);
				mCurrX = Math.max(mCurrX, mMinX);
				
				mCurrY = mStartY + Math.round(distanceCoef * (mFinalY - mStartY));
				// Pin to mMinY <= mCurrY <= mMaxY
				mCurrY = Math.min(mCurrY, mMaxY);
				mCurrY = Math.max(mCurrY, mMinY);
				if (mCurrX == mFinalX && mCurrY == mFinalY) {
					mFinished = true;
				}
				break;
			}
		}
		else {
			mCurrX = mFinalX;
			mCurrY = mFinalY;
			mFinished = true;
		}
		return true;
	}
	
	public void fling(int startX, int startY, int velocityX, int velocityY,
			int minX, int maxX, int minY, int maxY) {
		// Continue a scroll or fling in progress
		if (mFlywheel && !mFinished) {
			float oldVel = getCurrVelocity();
			float dx = (float) (mFinalX - mStartX);
			float dy = (float) (mFinalY - mStartY);
			float hyp = (float) Math.hypot(dx, dy);
			float ndx = dx / hyp;
			float ndy = dy / hyp;
			float oldVelocityX = ndx * oldVel;
			float oldVelocityY = ndy * oldVel;
			if (Math.signum(velocityX) == Math.signum(oldVelocityX) &&
					Math.signum(velocityY) == Math.signum(oldVelocityY)) {
				velocityX += oldVelocityX;
				velocityY += oldVelocityY;
			}
		}
		mMode = FLING_MODE;
		mFinished = false;
		float velocity = (float) Math.hypot(velocityX, velocityY);
	
		mVelocity = velocity;
		mDuration = getSplineFlingDuration(velocity);
		mStartTime = AnimationUtils.currentAnimationTimeMillis();
		mStartX = startX;
		mStartY = startY;
		float coeffX = velocity == 0 ? 1.0f : velocityX / velocity;
		float coeffY = velocity == 0 ? 1.0f : velocityY / velocity;
		double totalDistance = getSplineFlingDistance(velocity);
		mDistance = (int) (totalDistance * Math.signum(velocity));
		
		mMinX = minX;
		mMaxX = maxX;
		mMinY = minY;
		mMaxY = maxY;
		mFinalX = startX + (int) Math.round(totalDistance * coeffX);
		// Pin to mMinX <= mFinalX <= mMaxX
		mFinalX = Math.min(mFinalX, mMaxX);
		mFinalX = Math.max(mFinalX, mMinX);
		
		mFinalY = startY + (int) Math.round(totalDistance * coeffY);
		// Pin to mMinY <= mFinalY <= mMaxY
		mFinalY = Math.min(mFinalY, mMaxY);
		mFinalY = Math.max(mFinalY, mMinY);
	}
}
```

### OnDragListener

OnDragListener 是一个用于处理拖放事件的接口，在拖放时，它实际并不是将 View 移动，而是生成了一个被拖放 View 大小相同的像素进行移动，在结束拖动时，可以获取传递的数据，适合跨进程使用。

#### 使用方式

使用 OnDragListener 时需要完成步骤：

- 创建 OnDragListener 实例，并调用 View.setDragListener() 设置监听
- 调用 View.startDrag() 方法

```java
view.setOnDragListener(new OnDragListener() {
	@Override
	public boolean onDrag(View v, DragEvent event) {
		switch(event.getAction()) {
			case DragEvent.ACTION_DRAG_STARTED:
				// 拖动开始时触发
				break;
			case DragEvent.ACTION_DRAG_ENTERED:
              	  // 当拖动的对象进入 View 时触发
				break;
             case DragEvent.ACTION_DRAG_LOCATION:
                  // 当拖动的对象在 View 内移动时触发
                  break;
			case DragEvent.ACTION_DRAG_EXITED:
                  // 当拖动的对象离开 View 时触发
				break;	
			case DragEvent.ACTION_DRAG_ENDED:
                  // 拖动结束时触发
				break;
			case DragEvent.ACTION_DROP:
                  // 当拖动的对象在 View 上释放时触发
				break;		
		}
	}
});

DragShadowBuilder shadowBuilder = new View.DragShadowBuilder(v);
view.startDragAndDrop(null, shadowBuilder, view, 0);
```

startDragAndDrop(ClipData data, DragShadowBuilder shadowBuilder, Object myLocalState, int flags) 四个参数：

- ClipData：拖拽传递的数据，会在 DragEvent.ACTION_DROP 拖拽结束松手时才能获取到 ClipData 的数据

- DragShadowBuilder：在拖拽时生成View的半透明像素，可以观察跟随手指的拖拽状态
- myLocalState：可以用它传递本地数据，随时获取该本地数据，跨进程通信时会返回null
- flags：控制拖拽时的操作，一般传递0即可

### ViewDragHelper

ViewDragHelper 是一个用于处理拖放事件的辅助类，通过接管 onInterceptTouchEvent() 和 onTouchEvent() 提供不同情况下的接口简化拖动 View 逻辑处理。

#### 使用方式

使用 ViewDragHelper 时需要完成步骤：

- 使用 ViewDragHelper.create() 创建 ViewDragHelper 对象
- ViewDragHelper  实例接管 View 的 onInterceptTouchEvent() 和 onTouchEvent()
- 提供 ViewDragHelper.Callback 处理 View 的拖拽
- 重写 View.computeScroll() 方法，调用 ViewDragHelper.continueSettling() 计算，然后调用 postInvalidate() 开启下一次重绘

```java
public class DragHelperGridView extends ViewGroup {
    private ViewDragHelper mViewDragHelper;

    public DragHelperGridView(Context context, AttributeSet attrs) {
        super(context, attrs);
        mViewDragHelper = ViewDragHelper.create(this, new DragCallback());
    }

    @Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        return mViewDragHelper.shouldInterceptTouchEvent(ev);
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        mViewDragHelper.processTouchEvent(event);
        return true;
    }

    private class DragCallback extends ViewDragHelper.Callback {
        @Override
        public boolean tryCaptureView(@NonNull View view, int pointerId) {
            //判断view是否要拖动的view，返回true表示要触发拖拽，返回false表示不拖拽
            return true;
        }

        @Override
        public void onViewCaptured(@NonNull View capturedChild, int activePointerId) {
            //确认被拖动的是capturedChild，一般情况下是tryCaptureView()返回true的view
            //但可以通过调用ViewDragHelper.captureChildView回调这个方法，修改被拖动的view
        }
        
        @Override
        public int clampViewPositionHorizontal(@NonNull View child, int left, int dx) {
            //限制View在拖拽时水平方向的偏移
            //重写该方法返回该参数left表示水平方向拖动不干预限制拖拽
            return left;
        }

        @Override
        public int clampViewPositionVertical(@NonNull View child, int top, int dy) {
            //限制View在拖拽时垂直方向的偏移
            //重写该方法返回该参数top表示垂直方向拖动不干预限制拖拽
            return top;
        }

        @Override
        public void onViewPositionChanged(@NonNull View changedView, int left, int top, int dx, int dy) {
            //View的位置发生改变时触发
            //changedView是发生位置变化的View
            //left和top分别表示View新的左上角的x和y坐标
            //dx和dy表示View在x和y轴方向的位置变化量。
            super.onViewPositionChanged(changedView, left, top, dx, dy);
        }

        @Override
        public void onViewReleased(@NonNull View releasedChild, float xvel, float yvel) {
            //松开手指时被触发
        }

        @Override
        public void onViewDragStateChanged(int state) {
            //此方法在拖动状态发生改变时触发
            //STATE_IDLE：闲置状态，没有拖动
            //STATE_DRAGGING：拖动中
            //STATE_SETTLING：释放后View正在自动归位中
        }
        
        @Override
        public void onEdgeTouched(int edgeFlags, int pointerId) {
            //用户触摸到边缘时被触发，一般需要边缘触发的，在这里调用ViewDragHelper.captureChildView()
            //修改被拖动的view，然后会重新触发onViewCaptured()回调
            //通过setEdgeTrackingEnabled(EDGE_ALL)方法来设置边缘拖拽的方向
            //EDGE_ALL = EDGE_LEFT | EDGE_TOP | EDGE_RIGHT | EDGE_BOTTOM;
        }
        
        public void onEdgeDragStarted(int edgeFlags, int pointerId) {
            
        }
    }

    @Override
    public void computeScroll() {
        if (mViewDragHelper.continueSettling(true)) {
            ViewCompat.postInvalidateOnAnimation(this);
        }
    }
}
```

除了手动拖动的处理，ViewDragHelper 还提供了以下方法，用于自动滑动效果，需要注意的是这些方法并不会立即生效，而是会在下一帧的时候才开始执行：

- settleCapturedViewAt(int finalLeft, int finalTop)：被 Captured View 移动到指定的位置
- flingCapturedView(int minLeft, int minTop, int maxLeft, int maxTop)：被 Captured View 惯性滑动
- smoothSlideViewTo(View child, int finalLeft, int finalTop)：让某个子 View 滑动到某个位置

#### 原理分析

ViewDragHelper 的回调都是通过接管 onTouchEvent() 的 processTouchEvent() 方法实现的：

```java
public void processTouchEvent(@NonNull MotionEvent ev) {
    final int action = ev.getActionMasked();
    final int actionIndex = ev.getActionIndex();
    if (action == MotionEvent.ACTION_DOWN) {
        //DOWN事件重置状态
        cancel();
    }
    //添加速度监听器
    if (mVelocityTracker == null) {
        mVelocityTracker = VelocityTracker.obtain();
    }
    mVelocityTracker.addMovement(ev);
    
    switch (action) {
        case MotionEvent.ACTION_DOWN: {
            final float x = ev.getX();
            final float y = ev.getY();
            final int pointerId = ev.getPointerId(0);
            final View toCapture = findTopChildUnder((int) x, (int) y);
            //内部回调tryCaptureView()是否判断拖动该view
            //内部还会调用captureChildView() 回调onViewCaptured()确定最终拖动的View
            //内部还会调用setDragState() 回调onViewDragStateChanged()
            tryCaptureViewForDrag(toCapture, pointerId);
            
            final int edgesTouched = mInitialEdgesTouched[pointerId];
            if ((edgesTouched & mTrackingEdges) != 0) {
                //回调边缘触摸onEdgeTouched()
                mCallback.onEdgeTouched(edgesTouched & mTrackingEdges, pointerId);
            }
            break;
        }
        case MotionEvent.ACTION_POINTER_DOWN: {
            ...
            break;
        }
        case MotionEvent.ACTION_MOVE: {
            if (mDragState == STATE_DRAGGING) {
                // If pointer is invalid then skip the ACTION_MOVE.
                if (!isValidPointerForActionMove(mActivePointerId)) break;
                final int index = ev.findPointerIndex(mActivePointerId);
                final float x = ev.getX(index);
                final float y = ev.getY(index);
                final int idx = (int) (x - mLastMotionX[mActivePointerId]);
                final int idy = (int) (y - mLastMotionY[mActivePointerId]);
                //内部回调clampViewPositionHorizontal/clampViewPositionVertical
                //实际是调用offsetLeftAndRight/offsetTopAndBottom改变位置
                //最后回调onViewPositionChanged()
                dragTo(mCapturedView.getLeft() + idx, mCapturedView.getTop() + idy,
                saveLastMotion(ev);
            } else {
                final int pointerCount = ev.getPointerCount();
                for (int i = 0; i < pointerCount; i++) {
                    final int pointerId = ev.getPointerId(i);
                    if (!isValidPointerForActionMove(pointerId)) continue;
                    final float x = ev.getX(i);
                    final float y = ev.getY(i);
                    final float dx = x - mInitialMotionX[pointerId];
                    final float dy = y - mInitialMotionY[pointerId];
                    //回调onEdgeDragStarted()
                    reportNewEdgeDrags(dx, dy, pointerId);
                    if (mDragState == STATE_DRAGGING) {
                        break;
                    }
                    final View toCapture = findTopChildUnder((int) x, (int) y);
                    if (checkTouchSlop(toCapture, dx, dy)
                            && tryCaptureViewForDrag(toCapture, pointerId)) {
                        break;
                    }
                }
                saveLastMotion(ev);
            }
            break;
        }
        case MotionEvent.ACTION_POINTER_UP: {
            ...
            break;
        }
        case MotionEvent.ACTION_UP: {
            if (mDragState == STATE_DRAGGING) {
                //回调onViewReleased()
                //回调setDragState()
                releaseViewForPointerUp();
            }
            cancel();
            break;
        }
        case MotionEvent.ACTION_CANCEL: {
            if (mDragState == STATE_DRAGGING) {
                //回调onViewReleased()
                //回调setDragState()
                dispatchViewReleased(0, 0);
            }
            cancel();
            break;
        }
    }
}
                       
boolean tryCaptureViewForDrag(View toCapture, int pointerId) {
    if (toCapture == mCapturedView && mActivePointerId == pointerId) {
        // Already done!
        return true;
    }
    if (toCapture != null && mCallback.tryCaptureView(toCapture, pointerId)) {
        mActivePointerId = pointerId;
        captureChildView(toCapture, pointerId);
        return true;
    }
    return false;
}
                       
void setDragState(int state) {
    mParentView.removeCallbacks(mSetIdleRunnable);
    if (mDragState != state) {
        mDragState = state;
        mCallback.onViewDragStateChanged(state);
        if (mDragState == STATE_IDLE) {
            mCapturedView = null;
        }
    }
}
                       
private void dragTo(int left, int top, int dx, int dy) {
    int clampedX = left;
    int clampedY = top;
    final int oldLeft = mCapturedView.getLeft();
    final int oldTop = mCapturedView.getTop();
    if (dx != 0) {
        clampedX = mCallback.clampViewPositionHorizontal(mCapturedView, left, dx);
        ViewCompat.offsetLeftAndRight(mCapturedView, clampedX - oldLeft);
    }
    if (dy != 0) {
        clampedY = mCallback.clampViewPositionVertical(mCapturedView, top, dy);
        ViewCompat.offsetTopAndBottom(mCapturedView, clampedY - oldTop);
    }
    if (dx != 0 || dy != 0) {
        final int clampedDx = clampedX - oldLeft;
        final int clampedDy = clampedY - oldTop;
        mCallback.onViewPositionChanged(mCapturedView, clampedX, clampedY,
                clampedDx, clampedDy);
    }
}
                       
private void releaseViewForPointerUp() {
    mVelocityTracker.computeCurrentVelocity(1000, mMaxVelocity);
    final float xvel = clampMag(
            mVelocityTracker.getXVelocity(mActivePointerId),
            mMinVelocity, mMaxVelocity);
    final float yvel = clampMag(
            mVelocityTracker.getYVelocity(mActivePointerId),
            mMinVelocity, mMaxVelocity);
    dispatchViewReleased(xvel, yvel);
}
                       
private void dispatchViewReleased(float xvel, float yvel) {
    mReleaseInProgress = true;
    mCallback.onViewReleased(mCapturedView, xvel, yvel);
    mReleaseInProgress = false;
    if (mDragState == STATE_DRAGGING) {
        setDragState(STATE_IDLE);
    }
}
```

ViewDragHelper 提供的用于自动滑动效果的方法，其内部都是通过 Scroller 实现的：

```java
public boolean settleCapturedViewAt(int finalLeft, int finalTop) {
    return forceSettleCapturedViewAt(finalLeft, finalTop,
            (int) mVelocityTracker.getXVelocity(mActivePointerId),
            (int) mVelocityTracker.getYVelocity(mActivePointerId));
}

public boolean smoothSlideViewTo(@NonNull View child, int finalLeft, int finalTop) {
    mCapturedView = child;
    mActivePointerId = INVALID_POINTER;
    boolean continueSliding = forceSettleCapturedViewAt(finalLeft, finalTop, 0, 0);
    if (!continueSliding && mDragState == STATE_IDLE && mCapturedView != null) {
        mCapturedView = null;
    }
    return continueSliding;
}

private boolean forceSettleCapturedViewAt(int finalLeft, int finalTop, int xvel, int yvel) {
    final int startLeft = mCapturedView.getLeft();
    final int startTop = mCapturedView.getTop();
    final int dx = finalLeft - startLeft;
    final int dy = finalTop - startTop;
    if (dx == 0 && dy == 0) {
        // Nothing to do. Send callbacks, be done.
        mScroller.abortAnimation();
        setDragState(STATE_IDLE);
        return false;
    }
    final int duration = computeSettleDuration(mCapturedView, dx, dy, xvel, yvel);
    //settleCapturedViewAt内部使用Scroller，所以需要重写View.computeScroll()
    //调用ViewDragHelper.continueSettling()计算
    mScroller.startScroll(startLeft, startTop, dx, dy, duration);
    setDragState(STATE_SETTLING);
    return true;
}

public void flingCapturedView(int minLeft, int minTop, int maxLeft, int maxTop) {
    if (!mReleaseInProgress) {
        throw new IllegalStateException("Cannot flingCapturedView outside of a call to "
                + "Callback#onViewReleased");
    }
    mScroller.fling(mCapturedView.getLeft(), mCapturedView.getTop(),
            (int) mVelocityTracker.getXVelocity(mActivePointerId),
            (int) mVelocityTracker.getYVelocity(mActivePointerId),
            minLeft, maxLeft, minTop, maxTop);
    setDragState(STATE_SETTLING);
}
```





[Android手势检测——GestureDetector全面分析_android gesturedetector onscroll_炎之铠的博客-CSDN博客](https://blog.csdn.net/totond/article/details/77881180)

[安卓自定义View进阶-手势检测(GestureDetector) (gcssloop.com)](http://www.gcssloop.com/customview/gestruedector.html)

[安卓自定义View进阶-缩放手势检测(ScaleGestureDecetor) (gcssloop.com)](http://www.gcssloop.com/customview/scalegesturedetector.html)

[不再迷惑，也许之前你从未真正懂得 Scroller 及滑动机制_frank909的博客-CSDN博客](https://blog.csdn.net/briblue/article/details/73441698)

[Android 拖拽滑动（OnDragListener和ViewDragHelper）_setondraglistener_VincentWei95的博客-CSDN博客](https://blog.csdn.net/qq_31339141/article/details/107597055)

[神奇的 ViewDragHelper，让你轻松定制拥有拖拽能力的 ViewGroup_frank909的博客-CSDN博客](https://blog.csdn.net/briblue/article/details/73730386)
