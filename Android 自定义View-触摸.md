##### Q1：View 的位置体系和 MotionEvent 的位置体系？

View 和触碰事件 MotionEvent 有很多关于位置的相似方法名的 api

- View 的位置体系

  View 的位置由它的四个顶点来决定，分别对应于 View 的四个属性：left、top、right、bottom，其中 left、top 是 View 左上角的位置坐标，right、bottom 是 View 的右下角的位置坐标。需要注意的是，这里的取值是相对与父 View 的，可以通过 View 的 getXxx()方法来获取。

  从 Android 3.0 开始，View增加了另外几个参数：x、y、translationX、translationY。其中 x、y 是 View 左上角的坐标，translationX、translationY 是 View 左上角相对于父 View 的偏移量。这几个参数也是相对于父 View 的坐标，translationX、translationY 默认值是 0。可以通过 getXxx() 方法获取。

  在初始状态时，偏移量都是为 0 的，x、y 和 left、top 是相等的。但当 View 发生了偏移，top 和 left 等值不会发生变化，此时发生变化的是 x、y 、translationX、translationY，它们之间的换算关系是：

  ```java
  x = left + translationX
  y = top + translationY
  ```

- MotionEvent 的位置体系

  MotionEvent 是手指触碰屏幕后产生的事件，这个事件也是有其位置的。MotionEvent.getX/getY 获取的是触摸事件相对于当前 View 左上角的坐标。MotionEvent.getRawX/getRawY 获取的是触摸事件相对于屏幕左上角的坐标。

  TouchSlop 是系统能识别出的被认为是滑动的最小距离，可以通过 ViewConfiguration.get(getContext()).getScaledTouchSlop() 来获取这个和系统设备有关的常量。

##### Q2：单点触碰与多点触碰？

View 的触碰事件分为单点触碰和多点触碰，两者相比较来看，多点触碰多了记录手指的功能。先从如何判断一个触碰事件的类型熟悉。

###### 触碰类型

获取触碰类型的方法有两个：

- getAction()

  调用底层方法，获取到触碰事件的类型。

  ```java
  public final int getAction() {
      return nativeGetAction(mNativePtr);
  }
  ```

  常见的事件有：

  - ACTION_DOWN：手指初次触碰到屏幕触发
  - ACTION_UP：手指最后一个离开屏幕触发
  - ACTION_MOVE：手指在屏幕上移动触发
  - ACTION_CANCEL：事件先交给当前 View 后被上层拦截时触发
  - ACTION_OUTSIDE：手指不在控件区域时触发

- getActionMasked()

  ```java
  public static final int ACTION_MASK = 0xff;
  public final int getActionMasked() {
      return nativeGetAction(mNativePtr) & ACTION_MASK;
  }
  ```

  getAction() 方法返回一个 int 类型的值，共有 32 位（0x00000000），最低 8 位(0x000000**ff**) 表示事件类型，再往前的 8 位(0x0000**ff**00) 表示事件编号。所以 getActionMasked() 就是对 getAction() 的结果进行过滤，只取事件类型。除上面的所写的常见事件，还有：

  - ACTION_POINTER_DOWN：有非主要的手指按下(**即按下之前已经有手指在屏幕上**)。
  - ACTION_POINTER_UP：有非主要的手指抬起(**即抬起之后仍然有手指在屏幕上**)。

- getActionIndex()

  上面已经处理了多点触碰下，如何获取触碰事件的事件类型。本方法用来获取多点触碰事件的事件编号。

  ```java
  public static final int ACTION_POINTER_INDEX_MASK  = 0xff00;
  public static final int ACTION_POINTER_INDEX_SHIFT = 8;
  public final int getActionIndex() {
      return (nativeGetAction(mNativePtr) & ACTION_POINTER_INDEX_MASK) >> ACTION_POINTER_INDEX_SHIFT;
  }
  ```

  不过需要注意的时候，这个方法只在 ACTION_DOWN、ACTION_UP、（前面这两个获取到的编号肯定是 0）ACTION_POINTER_DOWN、ACTION_POINTER_UP 这几个事件中有效。因为在手指移动时，getAction() 返回值永远是 0x00000002，没有手指编号信息。

  而且通过本方法获取到 PointIndex 是会变化的，Index 变化有以下几个特点：

  1. 从 0 开始，自动正常。（ACTION_DOWN 时事件编码就为0）
  2. 如果之前落下的手指抬起，后面的手指 Index 就会减小。
  3. Index 的变化趋向于第一次落下的数值

- getPointerId(int pointerIndex)

  上面说到只有在手指按下和抬起时，才能拿到 Index，并且 Index 会发生变化。而 pointerId 是一个从手指按下到手指抬起都不会变换的值，用做手指的唯一值，不会受到其他手指抬起和落下的影响。

  ```java
  public final int getPointerId(int pointerIndex) {
      return nativeGetPointerId(mNativePtr, pointerIndex);
  }
  ```

- findPointerIndex(int pointerId)

  调用 getPointerId 我们可以从 pointIndex 查找到 pointId。而 pointIndex 除了 getAcionIndex 可以获取外，还可以调用 findPointerIndex 从 pointId 查找到 pointIndex。

##### Q3：触碰事件分发机制模型？

Activity 实现了 Window.Callback 接口，所以触碰事件在收集后，最先传给 Activity 的dispatchTouchEvent() 方法，然后通过 PhoneWindow 的 superDispatchTouchEvent() 传递给 DecorView，再由 DecorView 进入 ViewGroup/View 的触碰事件分发机制。如果所有的 View 都没有消费，那么触碰事件会再返回到 Activity.onTouchEvent 处理。

```java
public boolean dispatchTouchEvent(MotionEvent ev) {
    if (ev.getAction() == MotionEvent.ACTION_DOWN) {
        onUserInteraction();
    }
    if (getWindow().superDispatchTouchEvent(ev)) {
        return true;
    }
    return onTouchEvent(ev);
}
```

ViewGroup/View 的事件分发机制实际上是一个责任链模式，整个过程汇总涉及到三个重要方法：

1. dispatchTouchEvent()：用于事件的分发，如果事件传递给当前 View，那么此方法一定会被调用。返回结果表示是否消耗当前事件。
2. onInterceptTouchEvent()：仅 ViewGroup 有这个方法，在 dispatchTouchEvent() 中调用，用来判断当前 View 是否拦截当前事件。返回结果表示是否拦截当前事件。
3. onTouchEvent()：在 dispatchTouchEvent() 中调用，用来处理点击事件，返回结果表示是否消耗当前事件。

在 ViewGroup/View 中这三个方法的关系类似下面的伪代码：

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

##### Q3：ViewGroup 事件分发机制？

```java
public boolean dispatchTouchEvent(MotionEvent ev) {
    ...
    boolean handled = false;
    if (onFilterTouchEventForSecurity(ev)) {
        final int action = ev.getAction();
        final int actionMasked = action & MotionEvent.ACTION_MASK;
        // 此处通过 ACTION_MASK 获取事件类型，如果是 ACTION_DOWN 事件就刷新重置状态。
        if (actionMasked == MotionEvent.ACTION_DOWN) {
            cancelAndClearTouchTargets(ev);
            resetTouchState();
        }
        // 第一步，检查当前 ViewGroup 是否拦截事件
        // 一种情况是事件如果是 ACTION_DOWN 就检查 onInterceptTouchEvent
        // 一种情况是 mFirstTouchTarget 不为空（也就是子View处理了此事件序列），换句话说，如果 ViewGroup 之前拦截了事件，子View没有处理事件，那 mFirstTouchTarget 就是 null，就不会再去判断是否需要拦截了。
        final boolean intercepted;
        if (actionMasked == MotionEvent.ACTION_DOWN || mFirstTouchTarget != null) {
            // 上面第二种情况，子View成功处理了事件，mFirstTouchTarget != null，就会进入。如果子View调用了 requestDisallowInterceptTouchEvent 方法设置 FLAG_DISALLOW_INTERCEPT 为 true。就不会再判断当前 View 的onInterceptTouchEvent 的返回值了，而是直接设置为不拦截
            final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
            if (!disallowIntercept) {
                intercepted = onInterceptTouchEvent(ev);
                ev.setAction(action); // restore action in case it was changed
            } else {
                intercepted = false;
            }
        } else {
            // 上面两种情况之外，也就是当前 View 拦截了事件，那么后续的事件给当前 View 处理
            intercepted = true;
        }
        ...
        // 检查是否是取消事件
        final boolean canceled = resetCancelNextUpFlag(this) || actionMasked == MotionEvent.ACTION_CANCEL;
        final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
        TouchTarget newTouchTarget = null;
        boolean alreadyDispatchedToNewTouchTarget = false;
        // 不是取消事件且当前 View 不拦截事件，就会向子 View 传递
        if (!canceled && !intercepted) {
            if (actionMasked == MotionEvent.ACTION_DOWN
                    || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                    || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                ...
                final int childrenCount = mChildrenCount;
                if (newTouchTarget == null && childrenCount != 0) {
                    ...
                    final View[] children = mChildren;
                    // 遍历 ViewGroup 所有子View，判断View 是否能接收事件，能否接收事件由两点衡量
                    // 1. 子View 是否在播放动画
                    // 2. 事件的坐标是否在子View 的区域内。
                    for (int i = childrenCount - 1; i >= 0; i--) {
                        ...
                        // dispatchTransformedTouchEvent 的第三个入参如果是 null 就会调用当前ViewGroup 的 super.dispatchTouchEvent，而 ViewGroup 的super 就是 View。也就是当前ViewGroup 处理事件。
                        // 不为 null，会调用子View 的 dispatchTouchEvent 如果返回 true 表示已经消费了事件
                        if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                            ...
                            // addTouchTarget 会设置前面的变量 mFirstTouchTarget
                            newTouchTarget = addTouchTarget(child, idBitsToAssign);
                            alreadyDispatchedToNewTouchTarget = true;
                            break;
                        }
                        ...
                    }
                    ...
                }
                ...
            }
        }
        // 如果遍历后时间没有被处理，有两种可能：
        // 1. ViewGroup 没有View；2. 子View 在onTouchEvent 返回 false。
        // 这两种情况 ViewGroup 会自己处理点击事件。
        if (mFirstTouchTarget == null) {
            // dispatchTransformedTouchEvent 的第三个入参是 null，当前ViewGroup 处理事件。
            handled = dispatchTransformedTouchEvent(ev, canceled, null, TouchTarget.ALL_POINTER_IDS);
        } else {
            ...
        }
        ...
    }
    ...
    return handled;
}
```

##### Q4：View 事件分发机制？

上面分析了 ViewGroup 的dispatchTouchEvent 方法，在子 View 都没有处理事件的情况下，会调用 ViewGroup.super.dispatchTouchEvent() ，也就是 View 的dispatchTouchEvent。

```java
public boolean dispatchTouchEvent(MotionEvent event) {
    ...
    boolean result = false;
    if (mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onTouchEvent(event, 0);
    }
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

由源码可见，开发者设置的 onTouchListener 优先级高于 onTouchEvent。

```java
public boolean onTouchEvent(MotionEvent event) {
    final float x = event.getX();
    final float y = event.getY();
    final int viewFlags = mViewFlags;
    final int action = event.getAction();
    // 判断当前View 是否可点击，不可点击的View 不会消耗事件
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
    // TouchDelegate 是在不改变View大小的情况下，增加View的点击面积，内部点击逻辑和 View 的类似
    if (mTouchDelegate != null) {
        if (mTouchDelegate.onTouchEvent(event)) {
            return true;
        }
    }
    // 如果当前View 可点击，开始具体的处理，所有 ToolTip和!clickable 的分支都不分析
    if (clickable || (viewFlags & TOOLTIP) == TOOLTIP) {
        switch (action) {
            case MotionEvent.ACTION_UP:
                boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
                if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {
                    // 如果当前View 长按的事件在 postDelay 中还没有触发
                    // mHasPerformedLongPress 是 false
                    if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {
                        // 移除长按监听
                        removeLongPressCallback();
                        if (!focusTaken) {
                            // PerformClick 实现 Runnable，内部执行 performClick() 方法
                            // performClick 中如果 OnClickListener 不为空，就返回 true 拦截事件
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
                    // 如果当前View 在滚动的View 中，CheckForTap 继承 Runnable，
                    // 用于若干时间后注册长按点击监听 checkForLongClick
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
                    // 创建 CheckForLongPress 对象也是继承 Runnable
                    // 在若干时间后 postDelay 出去执行 performLongClick 并设置 mHasPerformedLongPress = true;
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

##### Q5：View 的滑动冲突解决？

常见的滑动冲突有三种场景：

1. 外部滑动和内部滑动的方向不一致
2. 外部滑动和内部滑动的方向一致
3. 以上两种情况的嵌套

- 外部滑动和内部滑动的方向不一致

  这种冲突情况可以根据滑动过程中的两个点的坐标判断是哪个方向的滑动，再决定由哪个 View 来拦截事件。通用的解决方案有两种：

  - 外部拦截法

    点击事件都先经过 父View 的拦截处理，如果需要就拦截，如果不需要就不拦截。

    1. 重写父 View 的 onInterceptTouchEvent() 方法，onInterceptTouchEvent() 中 ACTION_DOWN 返回 false，使得事件可以传递到子 View。如果一旦回 true，父 View 会直接拦截后续的事件，不会传递给子 View。
    2. onInterceptTouchEvent() 中 ACTION_MOVE 根据需要，如果父 View 拦截就返回 true，如果子 View 就返回 false。
    3. onInterceptTouchEvent() 中 ACTION_UP 返回 false。

  - 内部拦截法

    父 View 不拦截任何事件，所有的事件都先交给 子View ，如果子View 需要就直接消耗，否则就交给父View 处理。

    1. 重写父 View 的 onInterceptTouchEvent() 方法，onInterceptTouchEvent() 的 ACTION_DOWN 返回 false 不拦截，剩下的都返回 true 拦截。
    2. 重写子 View 的 dispatchTouchEvent() 方法，dispatchTouchEvent() 中 ACTION_DOWN 调用 parent.requestDisallowInterceptTouchEvent(true) 不允许父 View 拦截事件。
    3. 子 View 的 dispatchTouchEvent() 中 ACTION_MOVE ，如果父 View 需要就调用  parent.requestDisallowInterceptTouchEvent(false) 允许父 View 拦截事件，而父 View 的 onInterceptTouchEvent 中除了 ACTION_DOWN 都拦截了事件。

- 外部滑动和内部滑动的方向一致

  这种冲突情况 ，可以依靠业务上的逻辑规定，当某种状态需要哪个 View 来拦截事件。