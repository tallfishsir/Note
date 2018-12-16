Android 对手指在屏幕上的滑动有一个配置的最小距离，如果手指的滑动距离小于这个距离，系统就不会认为是在滑动。这个常量和设备有关，可以通过 ViewConfiguration.get(Context).getScaleTouchSlop() 来获取。

比较常见的滑动 View 的方法有：

##### scrollTo/scrollBy

###### mScrollX/mScrollY 

滑动时，View 内部有两个属性：mScrollX、mScrollY 用来描述 View 滑动的距离。

mScrollX 的值等于 View 内容的左边缘到 View 左边缘的距离。也就是说，如果 View 内容左边缘在 View 左边缘的左边时，mScrollX 是正值。

mScrollY 的值等于 View 内容的上边缘到 View 上边缘的距离。也就是说，如果 View 内容的上边缘在 View 的上边缘的上面时，mScrollY 是正值。

上面两个属性可以通道调用 getScrollX() getScrollY() 获取。

###### scrollTo/scrollBy

首先明确一点，这两个方法本质上改变的是 mScrollX 和 mSrollY 两个值。而这两个值代表的是 View 内容到 View 边界的距离。所以给开发者的感觉是，设置 mScroll 为正值，View 的内容反而向上移动。为了平时开发方便，可以暂时简便的认为：当使用 scrollTo/scrollBy 时，X 轴的正向向左，Y 轴的正向向上。

```java
// 实现了基于入参的绝对滑动，单位是像素
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

// 实现了基于当前位置的相对滑动
public void scrollBy(int x, int y) {
    scrollTo(mScrollX + x, mScrollY + y);
}
```

##### Scroller

上面的两个方法都是瞬时性的，而平常我们的滑动交互都是渐进式的。要实现这种效果的共同思想是将一个大的滑动分成若干个小的滑动在一段时间内完成。

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
```

Scroller 在使用时有个注意点：

1. 首先通过 startScroll(int startX, int startY, int dx, int dy, int duration) 开始滑动的准备，但这时候并没有滑动，从源码中可以看到，只是设置了一些属性。
2. invalidate() 会导致重绘，而在 View 的draw() 方法中会调用 computeScroll() 方法。
3. 重写 computeScroll()，并调用 computeScrollOffset()，这个方法是关键，它的返回值表明滑动有没有完成，方法内容又会计算出 mCurrX、mCurrY 的值。
4. 调用 View 的 scrollTo(mScroller.getCurrX(), mScroller.getCurrY()) 开始真正的滑动
5. 再次调用 postInvalidate() 刷新界面。

##### GestureDetector

##### View 的滑动冲突解决

常见的滑动冲突有三种场景：

1. 外部滑动和内部滑动的方向不一致
2. 外部滑动和内部滑动的方向一致
3. 以上两种情况的嵌套

###### 外部滑动和内部滑动的方向不一致

这种冲突情况可以根据滑动过程中的两个点的坐标判断是哪个方向的滑动，再决定由哪个 View 来拦截事件。通用的解决方案有两种：

- 外部拦截法

  点击事件都先经过 父View 的拦截处理，如果需要就拦截，如果不需要就不拦截。

  1. 重写 父View 的 onInterceptTouchEvent() 方法
  2. onInterceptTouchEvent() 中 ACTION_DOWN 返回 false，使得事件可以传递到 子View。否则一旦返回 true，父View 会直接拦截后续的事件，不会传递给子View。
  3. onInterceptTouchEvent() 中 ACTION_MOVE 根据需要，如果 父View 拦截就返回 true，如果 子View 就返回 false。
  4. onInterceptTouchEvent() 中 ACTION_UP 返回 false

- 内部拦截法

  父 View 不拦截任何事件，所有的事件都先交给 子View ，如果子View 需要就直接消耗点，否则就交给 父View 处理。

  1. 重写 父View 的 onInterceptTouchEvent() 方法
  2. 父View 的 onInterceptTouchEvent() 的 ACTION_DOWN 返回 false 不拦截，剩下的都返回 true 拦截。
  3. 重写 子View 的 dispatchTouchEvent() 方法
  4. 子View 的 dispatchTouchEvent() 中 ACTION_DOWN 调用 parent.requestDisallowInterceptTouchEvent(true) 不允许 父View 拦截事件。
  5. 子View 的 dispatchTouchEvent() 中 ACTION_MOVE ，如果 父View 需要就调用  parent.requestDisallowInterceptTouchEvent(false) 允许 父View拦截事件，而 父View 的 onInterceptTouchEvent 中除了 ACTION_DOWN 都拦截了事件。

###### 外部滑动和内部滑动的方向一致

这种冲突情况 ，可以依靠业务上的逻辑规定，当某种状态需要哪个 View 来拦截事件。