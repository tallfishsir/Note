# View 手势操作

## 手势检测 GestureDetector

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

## 缩放手势检测 ScaleGestureDetector

ScaleGestureDetector 的 Listener 有三个方法

- boolean onScaleBegin(ScaleGestureDetector detector) 缩放手势开始，当两个手指放在屏幕上的时候会调用该方法(只调用一次)，如果返回 false 则表示不使用当前这次缩放手势
- boolean onScaleBegin(ScaleGestureDetector detector) 缩放被触发(会调用0次或者多次)，如果返回 true 则表示当前缩放事件已经被处理，检测器会重新积累缩放因子，返回 false 则会继续积累缩放因子
- void onScaleEnd(ScaleGestureDetector detector) 缩放手势结束

ScaleGestureDetector 有三个重要方法

- getFocusX()：缩放中心 x 坐标
- getFocusY()：缩放中心 y 坐标
- getScaleFactor()：缩放因子

## 拖拽滑动

## 滑动机制

