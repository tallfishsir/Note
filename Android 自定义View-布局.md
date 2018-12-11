##### draw - drawBackground - onDraw - dispatchDraw  - onDrawForeground

##### animation

##### touchEvent - mulTouchEvent 

##### scroller - Gesture - viewDrawHelper

自定义 View 的全部过程包括 布局过程、绘制过程、动画、触摸反馈几大流程，其中布局过程涉及到 measure() 和 layout() 两个调度方法，对于不同的需求效果，粗略可以分为三种形式：

1. 修改已有 View 的尺寸，只需要重写 onMeasure() 来修改。
2. 定义全新的 View，需要重写 onMeasure() 来计算 View 的尺寸大小。
3. 定义全新的 ViewGroup，需要重写 onMeasure() 计算尺寸大小、重写 onLayout() 设置子 View 的位置。在 onMeasure() 中需要先计算所有子 View 的尺寸后再确定自身的大小，在 onLayout() 中需要先确定自身的位置再设置子 View 的位置。

View 的测量阶段是从 measure() 开始，measure() 被父 View 调用后，做一些准备优化工作后，调用 onMeasure 进行实际的自我测量。布局阶段 layout() 被父 View 调用，在 layout()  中会保存父 View 传进来的自己的位置和尺寸，然后调用 onLayout() 进行自己内部子 View 的布局，两者都是从上到下一层层迭代下来的。

##### Measure 阶段

View 的 onMeasure 方法

```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
            getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
}

// 获取 View 的默认大小，抛开 MeasureSpec.UNSPECIFIED 模式，返回的是父 View 对自己的尺寸要求
public static int getDefaultSize(int size, int measureSpec) {
    int result = size;
    // MeasureSpec.getMode 用于解析 measureSpec 的测量模式
    int specMode = MeasureSpec.getMode(measureSpec);
    // MeasureSpec.getSize 用于解析 measureSpec 的测量大小
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

在这个源码中我需要熟悉两件事，1. 入参 widthMeasureSpec/heightMeasureSpec 是什么；2. onMeasure 在测量结束前，必须用 setMeasuredDimension 方法将计算的宽高值设置给 View 才有效。

measureSpec 实际上是 View 的测量数据，是一个 int 类型的数据，长度 32 位。它的高 2 位是标识测量模式，有三种：MeasureSpec.EXACTLY、MeasureSpec.AT_MOST、MeasureSpec.UNSPECIFIED。它的低 30 位是实际测量的大小。准确点说，在 onMeasure 的入参中 measureSpec 代表的是父 View 对自己测量的要求。

前面所说自定义 View 第一种情况，我们需要修改 onMeasure 的结果，设置一个想要的值，但这个值可能和父 View 的要求不一致，这时候可以通过 resolveSizeAndState 来处理。

```java
// 抛开 MeasureSpec.UNSPECIFIED 这种父View 没有要求的情况
// 剩下的两中情况，都是取开发者的要求 size 和父 View 的要求 specSize 的较小值
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

自定义 View 前两种情况的操作对象基本是 View，如果是自定义 ViewGroup，还需要测量子 View 的大小，需要在 onMeasure 中对每一个子 View 调用 ViewGroup.measureChildWithMargins 

```java
// 类似于类加载过程中的委托模式，在父View的 onMeasure 中调用 child.measure 达到测量子View的目的
// 在子View测量完成后，可以调用 View.getMeasuredWidth/getMeasuredHeight 获取测量的值
protected void measureChildWithMargins(View child,
        int parentWidthMeasureSpec, int widthUsed,
        int parentHeightMeasureSpec, int heightUsed) {
    // MarginLayoutParams 获取的是开发者对View的布局要求，
    // >=0 是具体的值 -1=match_parent -2=wrap_content
    final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();
    // 通过开发者要求和父 View 布局要求一起计算出 measureSpec 值
    final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
            mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                    + widthUsed, lp.width);
    final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
            mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                    + heightUsed, lp.height);
    // 将前面计算出的 measureSpec 值传给子 View （对子View而言，这就是父View对它的布局要求）
    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}

// size 是父布局要求 - padding 的值，specMode 是父布局的测量模式要求
// childDimension 开发者的要求
// 有两种遍历方法，但我认为源码中的遍历方式不容易理解，在我看来，开发者的要求最大，最应该满足：
// childDimension > 0 → resultSize=childDimension resultMode=EXACTLY
// 开发者要求match_parent specMode=EXACTLY → resultSize=size resultMode=EXACTLY
// 开发者要求match_parent specMode=AT_MOST → resultSize=size resultMode=AT_MOST
// 开发者要求wrap_content specMode=EXACTLY → resultSize=size resultMode=AT_MOST
// 开发者要求wrap_content specMode=AT_MOST → resultSize=size resultMode=AT_MOST
// 最后调用 MeasureSpec.makeMeasureSpec 合成 resultSize 和 resultMode
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
    int specMode = MeasureSpec.getMode(spec);
    int specSize = MeasureSpec.getSize(spec);
    int size = Math.max(0, specSize - padding);
    int resultSize = 0;
    int resultMode = 0;
    switch (specMode) {
    // Parent has imposed an exact size on us
    case MeasureSpec.EXACTLY:
        if (childDimension >= 0) {
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size. So be it.
            resultSize = size;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size. It can't be
            // bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        }
        break;
    // Parent has imposed a maximum size on us
    case MeasureSpec.AT_MOST:
        if (childDimension >= 0) {
            // Child wants a specific size... so be it
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size, but our size is not fixed.
            // Constrain child to not be bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size. It can't be
            // bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        }
        break;
    // Parent asked to see how big we want to be
    case MeasureSpec.UNSPECIFIED:
        if (childDimension >= 0) {
            // Child wants a specific size... let him have it
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size... find out how big it should
            // be
            resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
            resultMode = MeasureSpec.UNSPECIFIED;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size.... find out how
            // big it should be
            resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
            resultMode = MeasureSpec.UNSPECIFIED;
        }
        break;
    }
    return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
}
```

##### Layout 布局阶段

```java
// 入参传入的 l\t\r\b 分别是自身在父View中的坐标，
// 如果自身的大小位置发生变化，会在 setFrame 中调用 onSizeChanged(newWidth, newHeight, oldWidth, oldHeight) 回调
public void layout(int l, int t, int r, int b) {
    if ((mPrivateFlags3 & PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT) != 0) {
        onMeasure(mOldWidthMeasureSpec, mOldHeightMeasureSpec);
        mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
    }
    int oldL = mLeft;
    int oldT = mTop;
    int oldB = mBottom;
    int oldR = mRight;
    boolean changed = isLayoutModeOptical(mParent) ?
            setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);
    if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
        onLayout(changed, l, t, r, b);
        mPrivateFlags &= ~PFLAG_LAYOUT_REQUIRED;
        ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnLayoutChangeListeners != null) {
            ArrayList<OnLayoutChangeListener> listenersCopy =
                    (ArrayList<OnLayoutChangeListener>)li.mOnLayoutChangeListeners.clone();
            int numListeners = listenersCopy.size();
            for (int i = 0; i < numListeners; ++i) {
                listenersCopy.get(i).onLayoutChange(this, l, t, r, b, oldL, oldT, oldR, oldB);
            }
        }
    }
    mPrivateFlags &= ~PFLAG_FORCE_LAYOUT;
    mPrivateFlags3 |= PFLAG3_IS_LAID_OUT;
}
```

当自身的大小位置确定后，就可以调用 onLayout() 确定子View 的布局。