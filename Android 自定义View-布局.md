##### Q1：View 工作的三大流程？

从 ActivityManagerService 和 WindowManagerService 的工作流程中可以总结出：ActivityThread 的 handleLaunchActivity 的方法中 Activity.attach() 创建了一个 PhoneWindow 对象，然后在 Activity.onCreate() 中 setContentView 创建 DecorView，最后在 ActivityThread.handleResumeActivity() 中完成了 Window 添加 View 的流程。添加 View 的流程中会调用 ViewRootImpl.requestLayout() → scheduleTraversals()，其中有一个 Runnable 类型的对象 TraversalRunnable，它的 run() 中会执行 doTraversal() → performTraversals()，ViewRootImpl.performTraversals() 就是 View 绘制流程的入口。

performTraversals() 方法会依次调用 performMeasure() 、performLayout()、performDraw()，这三个方法内部又会调用 DecorView 的 measure()、layout()、draw()，这三个方法又会调用 onMeasure()、onLayout()、onDraw()，这三个方法又会调用子 View 的 measure()、layout()、draw() 完成传递。

measure 过程确定了 View 的宽/高测试值，可以通过 getMeasuredWidth()/getMeasuredHeight() 获取测量值，layout 过程确定了 View 的四个顶点的坐标和实际的宽/高值，可以通过 getTop()/getBottom()/getLeft()/getRight() 获取，draw 过程确定了 View 的显示内容。

##### Q2：理解 MeasureSpec？

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
    @MeasureSpecMode
    public static int getMode(int measureSpec) {
        //noinspection ResourceType
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
- EXACTLY：父 View 已经计算出子 View 的大小
- AT_MOST：父 View 最后允许子 View 的大小

View 的 SpecMode 代表父 View 对测量的要求， LayoutParams 则代表着开发者对 View 的测量要求。

##### Q3：View 的 measure 过程？

View 的 measure 方法被 final 修饰，它会调用 View 的 onMeasure 方法，在 onMeasure 的最后要通过 setMeasuredDimension() 方法设置测量的值。

```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
            getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
}

// 如果没有背景返回默认的 0，否则取两者最大
protected int getSuggestedMinimumWidth() {
    return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
}

// 获取 View 的默认大小，除去 MeasureSpec.UNSPECIFIED 模式，返回的是父 View 对自己的尺寸要求
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

一般来说，setMeasuredDimension() 的入参是一个开发者计算出来的值，但这个值很有可能不符合父 View 的要求，所以可以通过 resolveSizeAndState()，进行一次矫正。

前面所说自定义 View 第一种情况，我们需要修改 onMeasure 的结果，设置一个想要的值，但这个值可能和父 View 的要求不一致，这时候可以通过 resolveSizeAndState 来处理。

```java
// MeasureSpec.UNSPECIFIED 不处理
// MeasureSpec.EXACTLY 取父 View 的要求
// MeasureSpec.AT_MOST 取开发者的要求和父 View 的要求的较小值
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

##### Q4：ViewGroup 的 measure 过程？

对于 ViewGroup 来说，处理完成自己的 measure，还需要遍历调用所有子元素的 measure 方法。ViewGroup 没有重写 onMeasure 方法，并且提供了 measureChildren() 来测量子 View，这个方法中会调用 measureChild()，还有一个很相似的方法是 measureChildWithMargins()，增加了 padding/margin 的情况。

```java
protected void measureChildWithMargins(View child,
        int parentWidthMeasureSpec, int widthUsed,
        int parentHeightMeasureSpec, int heightUsed) {
    // MarginLayoutParams 获取的是开发者对View的布局要求，
    // >=0 是具体的值 -1=match_parent -2=wrap_content
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
 * 有两种遍历方法，但我认为源码中的遍历方式不容易理解，在我看来，开发者的要求最大，最应该满足：
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

##### Q5：在 Activity 中获取 View 的宽/高信息方法？

Activity 的启动过程和 View 的加载是异步的，在 onCreate、onStart、onResume 中均无法正确得到某个 View 的宽高信息，有两种方法可以解决：

- Activity/View . onWindowFocusChanged

  onWindowFocusChanged 是在 View 的初始化已经完毕，在 Activity 的窗口得到焦点和失去焦点时都会被调用一次，具体的说，就是 Activity 的 onPause、onResume 执行时会被调用。

- View.post(Runnable)

  在 onStart 声明周期内，通过 post 将一个 Runnable 放入主线程的消息队列尾部，根据前面了解到的加载过程，View 的绘制会在这个 Runnable 之前执行。

##### Q6：View 的 layout 过程？

```java
/**
 * 入参传入的 l\t\r\b 分别是自身在父View中的坐标
 * 如果自身的大小位置发生变化，会在 setFrame 中调用 onSizeChanged(newWidth, newHeight, oldWidth, oldHeight) 回调
 */
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
            ArrayList<OnLayoutChangeListener> listenersCopy = (ArrayList<OnLayoutChangeListener>)li.mOnLayoutChangeListeners.clone();
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

##### Q7：addView 和 removeView 相似函数与相关函数

ViewGroup 对象可以通过 addView 和 removeView 来控制子 View，ViewGroup 内部有多个类似的函数，比如：

- attachViewToParent 和 detachViewFromParent
- addViewInLayout 和 removeViewInLayout
- addViewInner 和 removeViewInternal
- addInArray 和 removeFromArray

ViewGroup 内部是有一个数组 `private View[] mChildren`来保存子 View 的，有一个变量 `private int mChildrenCount`来保存子 View 的数量。

```java
private void addInArray(View child, int index) {
    View[] children = mChildren;
    final int count = mChildrenCount;
    final int size = children.length;
    if (index == count) {
        if (size == count) {
            mChildren = new View[size + ARRAY_CAPACITY_INCREMENT];
            System.arraycopy(children, 0, mChildren, 0, size);
            children = mChildren;
        }
        children[mChildrenCount++] = child;
    } else if (index < count) {
        if (size == count) {
            mChildren = new View[size + ARRAY_CAPACITY_INCREMENT];
            System.arraycopy(children, 0, mChildren, 0, index);
            System.arraycopy(children, index, mChildren, index + 1, count - index);
            children = mChildren;
        } else {
            System.arraycopy(children, index, children, index + 1, count - index);
        }
        children[index] = child;
        mChildrenCount++;
        if (mLastTouchDownIndex >= index) {
            mLastTouchDownIndex++;
        }
    } else {
        throw new IndexOutOfBoundsException("index=" + index + " count=" + count);
    }
}

private void removeFromArray(int index) {
    final View[] children = mChildren;
    if (!(mTransitioningViews != null && mTransitioningViews.contains(children[index]))) {
        children[index].mParent = null;
    }
    final int count = mChildrenCount;
    if (index == count - 1) {
        children[--mChildrenCount] = null;
    } else if (index >= 0 && index < count) {
        System.arraycopy(children, index + 1, children, index, count - index - 1);
        children[--mChildrenCount] = null;
    } else {
        throw new IndexOutOfBoundsException();
    }
    if (mLastTouchDownIndex == index) {
        mLastTouchDownTime = 0;
        mLastTouchDownIndex = -1;
    } else if (mLastTouchDownIndex > index) {
        mLastTouchDownIndex--;
    }
}
```

**addInArray** 和 **removeFromArray** 就是单纯的更新 ViewGroup 内的数组对象和子视图数量。

```java
protected boolean addViewInLayout(View child, int index, LayoutParams params, boolean preventRequestLayout) {
    ...
    addViewInner(child, index, params, preventRequestLayout);
    ...
    return true;
}
private void addViewInner(View child, int index, LayoutParams params, boolean preventRequestLayout) {
    ...
    addInArray(child, index);
    ...
    child.dispatchAttachedToWindow(mAttachInfo, mViewFlags&VISIBILITY_MASK));
    ...
}

public void removeViewInLayout(View view) {
    removeViewInternal(view);
}
private void removeViewInternal(int index, View view) {
    ...
    view.dispatchDetachedFromWindow();
    ...
    removeFromArray(index);
    ...
}
```

**addViewInLayout** 会调用 **addViewInner** 方法，这个方法中会调用 **addInArray** 来更新数组，然后调用子 View 的 dispatchAttachedToWindow 方法，这个方法最后会调用 View 的 onAttachedToWindow 回调。

**removeViewInLayout** 会调用 **removeViewInternal** 方法，这个方法中会调用 **removeFromArray** 来更新数组，然后调用子 View 的 dispatchDetachedFromWindow方法，这个方法最后会调用 View 的 onDetachedFromWindow 回调。

这组方法相比于上面的一组，相同的是会更新 ViewGroup 的子 View 数组信息，不同的是最终会回调 View 的 onAttachedToWindow/onDetachedFromWindow 方法。

```
public void addView(View child, int index, LayoutParams params) {
    ...
    requestLayout();
    invalidate(true);
    addViewInner(child, index, params, false);
}

public void removeViewAt(int index) {
    removeViewInternal(index, getChildAt(index));
    requestLayout();
    invalidate(true);
}
```

**addView** 会先调用 requestLayout 和 invalidate 方法来重新布局绘制 View，最后调用 **addViewInner** 方法。

**removeViewInLayout** 会先调用 **removeViewInternal** 方法，然后调用 requestLayout 和 invalidate 方法来重新布局绘制 View。

这组方法相比于上面的一组，相同的是会更新 ViewGroup 的子 View 数组信息，回调 View 的 onAttachedToWindow/onDetachedFromWindow 方法。不同的是会重新布局绘制 View。

LayoutInflater.Inflate(resId , null ) 只创建temp ,返回temp。不能正确处理宽和高是因为：layout_width 和 layout_height 是相对了父级设置的，必须与父级的 LayoutParams 一致。而此 temp 的 getLayoutParams 为 null。

LayoutInflater.Inflate(resId , parent, false )创建 temp，然后执行temp.setLayoutParams(params); 返回 temp。布局时会将宽高设置为 wrap_content，因为temp.setLayoutParams(params);这个params正是root.generateLayoutParams(attrs);得到的。

LayoutInflater.Inflate(resId , parent, true ) 创建temp，然后执行root.addView(temp, params);最后返回root