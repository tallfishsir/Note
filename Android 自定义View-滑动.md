##### Q1：View.scrollTo/scrollBy 如何实现滑动？

滑动时，View 内部有两个属性：mScrollX、mScrollY 用来描述 View 滑动的距离。mScrollX 的值等于 View 内容的左边缘到 View 左边缘的距离，mScrollY 的值等于 View 内容的上边缘到 View 上边缘的距离。这两个属性可以通道调用 getScrollX()/getScrollY() 获取。

在 View 的坐标系下，从计算关系来看：

```java
mScrollX  = View 左边缘坐标 - View 内容的左边缘坐标
mScrollY  = View 上边缘坐标 - View 内容的上边缘坐标
```

所以当 View 左边缘坐标和 View 上边缘坐标都是 0 时，如果 View 内容左边缘在 View 左边缘的左边时，mScrollX 是正值；如果 View 内容的上边缘在 View 的上边缘的上面时，mScrollY 是正值。

scrollTo/scrollBy 这两个方法本质上改变的是 mScrollX 和 mSrollY 两个值。也就是说这两个方法只能改变 View 内容的位置而不能改变 View 在布局中位置。scrollTo 实现了相对于父 View 的绝对滑动，scrollBy 内部调用了 scrollTo ，实现了相对于当前位置的滑动，两者单位都是像素。

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

##### Q2：View.layout/offsetLeftAndRight/offsetTopAndBottom 如何实现滑动？

在View进行绘制时，会调用onLayout()方法来设置显示的位置，因此，我们可以通过修改View的left、top、right、bottom四个属性来控制View的坐标。要控制View随手指滑动，因此需要在onTouchEvent()事件中进行滑动控制。

```java
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

而 offsetLeftAndRight/offsetTopAndBottom 两个方法是对于 layout 方法的封装。

##### Q3：Scroller/OverScroller 如何实现滑动？

scrollTo/scrollBy 都是瞬时性的，而平常我们的滑动交互都是渐进式的。要实现这种效果的共同思想是将一个大的滑动分成若干个小的滑动在一段时间内完成，Scrooller/OverScroller 就是用来帮助滑动的辅助计算类。

Scroller/OverScroller 调用 startScroll 方法开始计算滑动，但没有真正的滑动，内部只是设置一些初始值。

```java
Scroller mScroller = new Scroller(context);

private void smoothXScroller(int destX){
    int scrollX = getScrollX();
    int delta = destX - scrollX;
    mScroller.startScroll(scrollX, 0, delta, 0, 1000);
    invalidate();
}

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
```

然后在 View.computeScroll() 中调用 scrollTo 方法执行真正的滑动操作，并调用 postInvalidate() 进行下一次的刷新。invalidate() 会导致重绘，而在 View 的draw() 方法中会调用 computeScroll() 方法。

```java

@Override
public void computeScroll(){
    if(mScroller.computeScrollOffset()){
    	scrollTo(mScroller.getCurrX(), mScroller.getCurrY());
        postInvalidate();
    }
}

public boolean computeScrollOffset() {
    if (mFinished) {
        return false;
    }
    int timePassed = (int)(AnimationUtils.currentAnimationTimeMillis() - mStartTime);
    if (timePassed < mDuration) {
        switch (mMode) {
        case SCROLL_MODE:
            final float x = mInterpolator.getInterpolation(timePassed * mDurationReciproca
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
            mCurrX = Math.min(mCurrX, mMaxX);
            mCurrX = Math.max(mCurrX, mMinX);
            
            mCurrY = mStartY + Math.round(distanceCoef * (mFinalY - mStartY));
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
```

OverScroller 比 Scroller 出现的晚，大部分的功能是相同的，增加了对超出滑动边界的情况的处理。

##### Q4：OnDragListener 如何实现滑动？

View.setOnDragListener() 把拖动事件监听器对象设置给一个View对象。拖动监听事件需要实现 View.OnDragListenter 接口

```java
public interface OnDragListener {
    boolean onDrag(View v, DragEvent event);
}
```

##### Q5：ViewDraghelper 如何实现滑动？

ViewDragHelper是针对 ViewGroup 中的拖拽和重新定位 views 操作时提供了一系列非常有用的方法和状态追踪。它需要重写 ViewGroup 的 onInterceptTouchEvent() onTouchEvent() computeScroll() 方法：

```java
// 创建一个 ViewDragHeloper 对象
ViewDragHelper dragHelper = ViewDragHelper.create(this, new DragCallback());
@Override
public boolean onInterceptTouchEvent(MotionEvent ev) {
    // 处理事件分发，将ViewGroup的事件分发委托给ViewDragHelper处理
    return dragHelper.shouldInterceptTouchEvent(ev);
}
@Override
public boolean onTouchEvent(MotionEvent event) {
    // 处理触碰事件
    dragHelper.processTouchEvent(event);
    return true;
}
@Override
public void computeScroll() {
    // ViewDragHelper.continueSettling 类似于 Scroller.computeScrollOffset
    if (dragHelper.continueSettling(true)) {
        ViewCompat.postInvalidateOnAnimation(this);
    }
}
```

ViewDragHelper 有三个 API 都是用于让某个 View 滑动的：

```java
/**
 * 某个View自动滚动到指定的位置，初速度为0，可在任何地方调用，动画移动会回调 continueSettling 方法，直到结束
 */
public boolean smoothSlideViewTo(View child, int finalLeft, int finalTop)
/**
 * 以松手前的滑动速度为初值，让捕获到的子View自动滚动到指定位置，只能在Callback的onViewReleased()中使用，其余同上
 */
public boolean settleCapturedViewAt(int finalLeft, int finalTop)
/**
 * 以松手前的滑动速度为初值，让捕获到的子View在指定范围内fling惯性运动，只能在Callback的onViewReleased()中使用，其余同上
 */
public void flingCapturedView(int minLeft, int minTop, int maxLeft, int maxTop)
```

ViewDragHelper.Callback 是在 ViewDragHelper.create 时传入的监听事件

```java
public abstract static class Callback {
    /**
     * 状态改变的回调（STATE_IDLE,STATE_DRAGGING,STATE_SETTLING）
     */
    public void onViewDragStateChanged(int state) {}
    /**
     * 拖拽的View位置变化时回调，changedView为位置变化的view，left、top变化后的x、y坐标，dx、dy为新位置与旧位置的偏移量
     */
    public void onViewPositionChanged(View changedView, int left, int top, int dx, int dy) {}
    /**
     * 成功捕获到子View时或者手动调用captureChildView()时回调
     */
    public void onViewCaptured(View capturedChild, int activePointerId) {}
    /**
     * 当前拖拽的view松手或者ACTION_CANCEL时调用，xvel、yvel为离开屏幕时的速率
     */
    public void onViewReleased(View releasedChild, float xvel, float yvel) {}
    /**
     * 当触摸到边界时回调
     */
    public void onEdgeTouched(int edgeFlags, int pointerId) {}
    /**
     * true的时候会锁住当前的边界，false则unLock。锁定后的边缘就不会回调 onEdgeDragStarted()
     */
    public boolean onEdgeLock(int edgeFlags) {
        return false;
    }
    /**
     * ACTION_MOVE且没有锁定边缘时触发，在此可手动调用ViewDragHelper.captureChildView()触发从边缘拖动子View
     */
    public void onEdgeDragStarted(int edgeFlags, int pointerId) {}
    /**
     * 寻找当前触摸点View时回调此方法，如需改变遍历子view顺序可重写此方法
     */
    public int getOrderedChildIndex(int index) {
        return index;
    }
    /**
     * 返回拖拽子View在水平方向上被认为是拖动的距离，默认是0，若是默认值，则会在checkTouchSlop()中TouchSlop代替
     */
    public int getViewHorizontalDragRange(View child) {
        return 0;
    }
    /**
     * 返回拖拽子View在垂直方向上被认为是拖动的距离，默认是0，若是默认值，则会在checkTouchSlop()中TouchSlop代替
     */
    public int getViewVerticalDragRange(View child) {
        return 0;
    }
    /**
     * 对触摸view判断，如果需要当前触摸的子View进行拖拽移动就返回true，否则返回false
     */
    public abstract boolean tryCaptureView(View child, int pointerId);
    /**
     * 拖拽的子View在水平方向上移动的位置，child为拖拽的子View，left为子view应该到达的x坐标，dx为挪动差值
     */
    public int clampViewPositionHorizontal(View child, int left, int dx) {
        return 0;
    }
    /**
     * 拖拽的子View在垂直方向上移动的位置，child为拖拽的子View，left为子view应该到达的x坐标，dx为挪动差值
     */
    public int clampViewPositionVertical(View child, int top, int dy) {
        return 0;
    }
}
```

