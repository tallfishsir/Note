# View 刷新流程

ViewRootImpl 的 performTraversals 方法中开始执行 View 体系的测量布局绘制流程：

```java
private void performTraversals() {
    // DecorView 赋值为局部变量 host
    final View host = mView;
    ...
    // 执行 measure 逻辑
    boolean layoutRequested = mLayoutRequested && (!mStopped || mReportNextDraw);
    if (layoutRequested) {
        windowSizeMayChange |= measureHierarchy(host, lp, res, desiredWindowWidth, desiredWindowHeight);
    }
    ...
    // 执行 layout 逻辑
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

在整个 View 的刷新流程中，有多个标志位是通过二进制位运算来完成的：

```java
private static final int FLAG = 1;  // 实际值是：0001 = 1
private static final int FLAG_1 = FLAG << 1; // 实际值是：0010 = 2
private static final int FLAG_2 = FLAG << 2; // 实际值是：0100 = 4
private static final int FLAG_3 = FLAG << 3; // 实际值是：1000 = 8
```

当需要给标志位增加某个标志，就让 mFlag 与 标志位做或运算

```
mFlag |= FLAG_1
```

当需要给标志位去除某个标志，就让 mFlag 与标志位取反做与运算

```
mFlag &= ~FLAG_1
```

当需要判断标志位是否含有某个标志位，就让 mFlag 与标志位做与运算后比较

```
if（(mFlag & FLAG_1) == FLAG_1）
```

## measure

measure 过程完成了测量逻辑，确定了 View 的宽和高。

### MeasureSpec

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
- EXACTLY：父 View 已经计算出子 View 的确定的大小
- AT_MOST：父 View 最多允许子 View 的大小

### ViewRootImpl measureHierarchy

在 ViewRootImpl 的 performTraversals 方法中调用 measureHierarchy 开启了测量流程。并在不同的条件下，利用 getRootMeasureSpec() 方法完成了整个 View 测量过程中顶层 MeasureSpec 对象的赋值。

```java
// desiredWindowWidth/desiredWindowHeight是Window外部的限制宽高，可能是屏幕宽度也可能是Window宽高
// lp是Window自身布局对宽高的要求
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

private static int getRootMeasureSpec(int windowSize, int rootDimension) {
    int measureSpec;
    switch (rootDimension) {
    case ViewGroup.LayoutParams.MATCH_PARENT:
        // Window can't resize. Force root view to be windowSize.
        measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.EXACTLY);
        break;
    case ViewGroup.LayoutParams.WRAP_CONTENT:
        // Window can resize. Set max size for root view.
        measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.AT_MOST);
        break;
    default:
        // Window wants to be an exact size. Force root view to be that size.
        measureSpec = MeasureSpec.makeMeasureSpec(rootDimension, MeasureSpec.EXACTLY);
        break;
    }
    return measureSpec;
}
```

在 performMeasure 方法中，会将顶层 MeasureSpec 对象传给 ViewRootImpl 的 mView 对象，开启 View 的测量流程。

### View 默认 onMeasure

在 ViewRootImpl 的 measureHierarchy 方法中，调用 View 的 measure 方法开启测量。

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

### ViewGroup 默认 onMeasure

对于 ViewGroup 来说，除了处理完成自己的 measure，还需要遍历调用所有子元素的 measure 方法。

但 ViewGroup 没有重写 onMeasure 方法，而是提供了 measureChildren()/measureChildWithMargins() 来测量子 View。在测量过程中，实际上有两个对宽高的限制要求：

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
>  2、将为子布局生成的测量结果传递给子布局，子布局进行1步骤。这是个递归的过程，当遇到的子布局是View时，递归结束，开始回溯
>  3、根据子布局自己测量后的结果，结合父布局给自己的测量结果，记录下自己的测量值，至此一个 ViewGroup 测量完毕

## layout

layout 过程完成了布局逻辑，确定了 View 的位置。

### ViewRootImpl performLayout

在 ViewRootImpl 的 performTraversals 方法中调用 performLayout 开启了布局流程。其中调用 mView 的 layout 过程根据 measure 的结果，确定了 View 的摆放位置

```java
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

// 入参传入的 l\t\r\b 分别是自身在父View中的坐标
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

### 获取 View 的尺寸位置方法

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
|                                           |                                          |            |

## draw

draw  过程完成了绘制逻辑，确定了 View 的内容，将内容绘制到 layout 确定的区域。

### ViewRootImpl performDraw

```java
private void performDraw() {
    ...
    final boolean fullRedrawNeeded = mFullRedrawNeeded || mReportNextDraw;
    mIsDrawing = true;
    ...
    try {
        boolean canUseAsync = draw(fullRedrawNeeded);
        ...
    } fially {
        mIsDrawing = false;
    }
    ...
}

private boolean draw(boolean fullRedrawNeeded) {
    ...
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
    ...
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

不管是硬件加速绘制还是软件绘制，都会设置重绘的矩形区域，对于硬件加速绘制来说，重绘的区域为整个 Window 的大小，对于软件绘制来说是设置相交的矩形区域。

硬件绘制的 Canvas 对象每个都是重新生成的，Canvas类型为：RecordingCanvas。

软件绘制的 Canvas 对象通过 lockCanvas 生成后，会传递给所有的子布局，所有 View树 共享一个 Canvas 对象，Canvas类型为：CompatibleCanvas。

### View onDraw

View 的绘制统一在 draw() 方法中调度，最先开始的是 drawBackground() 它负责绘制 View 的背景，接着就会调用 onDraw() 方法来完成 View 本体的绘制，然后通过调用 dispatchDraw() 方法通知子 View 开始绘制，最后会调用 onDrawForeground() 绘制前景：

```java
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

有一些地方，ViewGroup 没有背景时会绕过 draw 而直接调用 dispatchDraw 方法，所以 ViewGroup 绘制往往是写在 dispatchDraw() 中。ViewGroup 默认会开启一个标志位 willNotDraw，如果需要 ViewGroup 也走 onDraw 方法，需要调用 setWillNotDraw() 来设置关闭。

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

重写 View/ViewGroup 的 isChildrenDrawingOrderEnabled() 方法，可以修改子 View 的绘制顺序。

## View 主动刷新方法

### invalidate

View.invalidate() 表示当前绘制内容无效，需要重新绘制。

```java
public void invalidate() {
    invalidate(true);
}

public void invalidate(boolean invalidateCache) {
    //invalidateCache 使绘制缓存失效
    invalidateInternal(0, 0, mRight - mLeft, mBottom - mTop, invalidateCache, true);
}


void invalidateInternal(int l, int t, int r, int b, boolean invalidateCache, boolean fullInvalidate) {
    ...
    //设置了跳过绘制标记
    if (skipInvalidate()) {
        return;
    }

    //PFLAG_DRAWN在draw()中被标识，表示此前该View已经绘制过
    //PFLAG_HAS_BOUNDS在setFrame()中被标识，表示该View已经layout过，确定过坐标了
    if ((mPrivateFlags & (PFLAG_DRAWN | PFLAG_HAS_BOUNDS)) == (PFLAG_DRAWN | PFLAG_HAS_BOUNDS)
            || (invalidateCache && (mPrivateFlags & PFLAG_DRAWING_CACHE_VALID) == PFLAG_DRAWING_CACHE_VALID)
            || (mPrivateFlags & PFLAG_INVALIDATED) != PFLAG_INVALIDATED
            || (fullInvalidate && isOpaque() != mLastIsOpaque)) {
        if (fullInvalidate) {
            //默认true
            mLastIsOpaque = isOpaque();
            //清除绘制标记
            mPrivateFlags &= ~PFLAG_DRAWN;
        }

        //需要绘制
        mPrivateFlags |= PFLAG_DIRTY;

        if (invalidateCache) {
            //1、加上绘制失效标记
            //2、清除绘制缓存有效标记
            //这两标记在硬件加速绘制分支用到
            mPrivateFlags |= PFLAG_INVALIDATED;
            mPrivateFlags &= ~PFLAG_DRAWING_CACHE_VALID;
        }
        
        final AttachInfo ai = mAttachInfo;
        final ViewParent p = mParent;
        if (p != null && ai != null && l < r && t < b) {
            final Rect damage = ai.mTmpInvalRect;
            //记录需要重新绘制的区域 damge，该区域为该View尺寸
            damage.set(l, t, r, b);
            //p 为该View的父布局，调用父布局的invalidateChild
            p.invalidateChild(this, damage);
        }
        ...
    }
}
```

当前要刷新的 View 确定了刷新区域后，就调用父布局的 invalidateChild() 方法，方法内区分了硬件加速绘制和软件绘制，最终调用到 ViewRootImpl  的方法

```java
//ViewGroup.java
public final void invalidateChild(View child, final Rect dirty) {
    final AttachInfo attachInfo = mAttachInfo;
    if (attachInfo != null && attachInfo.mHardwareAccelerated) {
        //如果是支持硬件加速，则走该分支
        onDescendantInvalidated(child, child);
        return;
    }
    //软件绘制
    ViewParent parent = this;
    if (attachInfo != null) {
        ...
        do {
            View view = null;
            if (parent instanceof View) {
                view = (View) parent;
            }
            ...
            parent = parent.invalidateChildInParent(location, dirty);
        } while (parent != null);
    }
}
```

硬件绘制分支 onDescendantInvalidated() 会向上遍历父布局，并将父布局中的 PFLAG_DRAWING_CACHE_VALID 标记清空，也就是绘制缓存清空。最终 ViewRootImpl 对象调用 scheduleTraversals() 开启刷新流程。

```java
//View.java
public void onDescendantInvalidated(@NonNull View child, @NonNull View target) {
    mPrivateFlags |= (target.mPrivateFlags & PFLAG_DRAW_ANIMATION);
    
    if ((target.mPrivateFlags & ~PFLAG_DIRTY_MASK) != 0) {
       //此处都会走
        mPrivateFlags = (mPrivateFlags & ~PFLAG_DIRTY_MASK) | PFLAG_DIRTY;
        //清除绘制缓存有效标记
        mPrivateFlags &= ~PFLAG_DRAWING_CACHE_VALID;
    }
    
    if (mLayerType == LAYER_TYPE_SOFTWARE) {
        //如果是开启了软件绘制，则加上绘制失效标记
        mPrivateFlags |= PFLAG_INVALIDATED | PFLAG_DIRTY;
        //更改target指向
        target = this;
    }

    if (mParent != null) {
        //调用父布局的onDescendantInvalidated
        mParent.onDescendantInvalidated(this, target);
    }
}

//ViewRootImpl.java
public void onDescendantInvalidated(@NonNull View child, @NonNull View descendant) {
    if ((descendant.mPrivateFlags & PFLAG_DRAW_ANIMATION) != 0) {
        mIsAnimating = true;
    }
    invalidate();
}

void invalidate() {
    //mDirty 为脏区域，也就是需要重绘的区域，mWidth，mHeight 为Window尺寸
    mDirty.set(0, 0, mWidth, mHeight);
    if (!mWillDrawSoon) {
        //开启View 三大流程
        scheduleTraversals();
    }
}
```

软件绘制分支 invalidateChildInParent() 会向上遍历父布局，并将父布局中的 PFLAG_DRAWING_CACHE_VALID 标记清空，也就是绘制缓存清空。最终 ViewRootImpl 对象调用 scheduleTraversals() 开启刷新流程。

```java
//ViewGroup.java
public ViewParent invalidateChildInParent(final int[] location, final Rect dirty) {
    //dirty 为失效的区域，也就是需要重绘的区域
    if ((mPrivateFlags & (PFLAG_DRAWN | PFLAG_DRAWING_CACHE_VALID)) != 0) {
        //该View绘制过或者绘制缓存有效
        if ((mGroupFlags & (FLAG_OPTIMIZE_INVALIDATE | FLAG_ANIMATION_DONE))
                != FLAG_OPTIMIZE_INVALIDATE) {
            //修正重绘的区域
            dirty.offset(location[CHILD_LEFT_INDEX] - mScrollX,
                    location[CHILD_TOP_INDEX] - mScrollY);
            if ((mGroupFlags & FLAG_CLIP_CHILDREN) == 0) {
                //如果允许子布局超过父布局区域展示
                //则该dirty 区域需要扩大
                dirty.union(0, 0, mRight - mLeft, mBottom - mTop);
            }
            final int left = mLeft;
            final int top = mTop;
            if ((mGroupFlags & FLAG_CLIP_CHILDREN) == FLAG_CLIP_CHILDREN) {
                //默认会走这，如果不允许子布局超过父布局区域展示，则取相交区域
                if (!dirty.intersect(0, 0, mRight - left, mBottom - top)) {
                    dirty.setEmpty();
                }
            }
            //记录偏移，用以不断修正重绘区域，使之相对计算出相对屏幕的坐标
            location[CHILD_LEFT_INDEX] = left;
            location[CHILD_TOP_INDEX] = top;
        } else {
            ...
        }
        //标记缓存失效
        mPrivateFlags &= ~PFLAG_DRAWING_CACHE_VALID;
        if (mLayerType != LAYER_TYPE_NONE) {
            //如果设置了缓存类型，则标记该View需要重绘
            mPrivateFlags |= PFLAG_INVALIDATED;
        }
        //返回父布局
        return mParent;
    }
    return null;
}

//ViewRootImpl.java
public ViewParent invalidateChildInParent(int[] location, Rect dirty) {
    checkThread();
    if (DEBUG_DRAW) Log.v(mTag, "Invalidate child: " + dirty);
    if (dirty == null) {
        //脏区域为空，则默认刷新整个窗口
        invalidate();
        return null;
    } else if (dirty.isEmpty() && !mIsAnimating) {
        return null;
    }
    ...
    invalidateRectOnScreen(dirty);
    return null;
}

private void invalidateRectOnScreen(Rect dirty) {
    final Rect localDirty = mDirty;
    //合并脏区域，取并集
    localDirty.union(dirty.left, dirty.top, dirty.right, dirty.bottom);
    ...
    if (!mWillDrawSoon && (intersected || mIsAnimating)) {
        //开启View的三大绘制流程
        scheduleTraversals();
    }
}
```

### postInvalidate

因为 ViewRootImpl 的 invalidateChildInParent 方法中调用了 checkThread()，invalidate() 只能在主线程调用，所以在子线程中开启刷新流程，需要调用 postInvalidate()。

```java
//View.java
public void postInvalidate() {
    postInvalidateDelayed(0);
}

public void postInvalidateDelayed(long delayMilliseconds) {
    final AttachInfo attachInfo = mAttachInfo;
    if (attachInfo != null) {
        //还是靠ViewRootImpl
        attachInfo.mViewRootImpl.dispatchInvalidateDelayed(this, delayMilliseconds);
    }
}

//ViewRootImpl.java
public void dispatchInvalidateDelayed(View view, long delayMilliseconds) {
    //此处Message.obj = view
    Message msg = mHandler.obtainMessage(MSG_INVALIDATE, view);
    mHandler.sendMessageDelayed(msg, delayMilliseconds);
}

public void handleMessage(Message msg) {
    switch (msg.what) {
        case MSG_INVALIDATE:
            //obj 即为待刷新的View
            ((View) msg.obj).invalidate();
            break;
            ...
    }
}
```

### requestLayout

requestLayout 向上遍历父布局，给每个布局设置 PFLAG_FORCE_LAYOU T和 PFLAG_INVALIDATED 标记，直到ViewRootImpl 对象调用 scheduleTraversals() 开启刷新流程。

```java
//View.java
public void requestLayout() {
    //清空测量缓存
    if (mMeasureCache != null) mMeasureCache.clear();
    ...
    //添加强制layout 标记，该标记触发layout
    mPrivateFlags |= PFLAG_FORCE_LAYOUT;
    //添加重绘标记
    mPrivateFlags |= PFLAG_INVALIDATED;
    //如果布局没有结束就不会布局 
    if (mParent != null && !mParent.isLayoutRequested()) {
        //如果上次的layout 请求已经完成
        //父布局继续调用requestLayout
        mParent.requestLayout();
    }
    ...
}

//PFLAG_FORCE_LAYOUT会在layout()中移除
public boolean isLayoutRequested() {
    return (mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT;
}

//ViewRootImpl.java
public void requestLayout() {
    //是否正在进行layout过程
    if (!mHandlingLayoutInLayoutRequest) {
        //检查线程是否一致
        checkThread();
        //标记有一次layout的请求
        mLayoutRequested = true;
        //开启View 三大流程
        scheduleTraversals();
    }
}
```

## 布局加载 

### LayoutInflater

获取 LayoutInflater 实例

```java
LayoutInflater layoutInflater = LayoutInflater.from(context);

public static LayoutInflater from(Context context) {
    LayoutInflater LayoutInflater =
            (LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
    if (LayoutInflater == null) {
        throw new AssertionError("LayoutInflater not found.");
    }
    return LayoutInflater;
}
```

inflate 布局问加你

```java
public View inflate(@LayoutRes int resource, @Nullable ViewGroup root) {
    return inflate(resource, root, root != null);
}

public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) {
    final Resources res = getContext().getResources();
    View view = tryInflatePrecompiled(resource, res, root, attachToRoot);
    if (view != null) {
        return view;
    }
    XmlResourceParser parser = res.getLayout(resource);
    try {
        return inflate(parser, root, attachToRoot);
    } finally {
        parser.close();
    }
}

public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
    synchronized (mConstructorArgs) {
        final Context inflaterContext = mContext;
        final AttributeSet attrs = Xml.asAttributeSet(parser);
        Context lastContext = (Context) mConstructorArgs[0];
        mConstructorArgs[0] = inflaterContext;
        View result = root;
        try {
            advanceToRootNode(parser);
            final String name = parser.getName();
            
            final View temp = createViewFromTag(root, name, inflaterContext, attrs);
            ViewGroup.LayoutParams params = null;
            if (root != null) {
            	params = root.generateLayoutParams(attrs);
            	if (!attachToRoot) {
            		temp.setLayoutParams(params);
            	}
            }
            rInflateChildren(parser, temp, attrs, true);
            if (root != null && attachToRoot) {
            	root.addView(temp, params);
            }
        }
        return result;
    }
}
```

inflate 使用 pull 来解析 xml 文件，然后调用 createViewFromTag() 创建出根 View，然后调用 rInflateChildren() 方法循环遍历这个根布局下的子元素。

```java
View createViewFromTag(View parent, String name, Context context, AttributeSet attrs,
        boolean ignoreThemeAttr) {
    ...
    try {
        View view = tryCreateView(parent, name, context, attrs);
        if (view == null) {
            final Object lastContext = mConstructorArgs[0];
            mConstructorArgs[0] = context;
            try {
                if (-1 == name.indexOf('.')) {
                    view = onCreateView(context, parent, name, attrs);
                } else {
                    view = createView(context, name, null, attrs);
                }
            } finally {
                mConstructorArgs[0] = lastContext;
            }
        }
        return view;
    }
    ...
}

public final View tryCreateView(@Nullable View parent, @NonNull String name,
    @NonNull Context context,
    @NonNull AttributeSet attrs) {
    View view;
    if (mFactory2 != null) {
        view = mFactory2.onCreateView(parent, name, context, attrs);
    } else if (mFactory != null) {
        view = mFactory.onCreateView(name, context, attrs);
    } else {
        view = null;
    }
    if (view == null && mPrivateFactory != null) {
        view = mPrivateFactory.onCreateView(parent, name, context, attrs);
    }
    return view;
}
```

在createViewFromTag() 中会调用 tryCreateView() 进行三次拦截最终调用系统方法生成 View，其中开发者可以在 Activity 的 onCreate() 中对 mFactory2（通过setFactory2进行赋值 ）和mFactory（通过setFactory进行赋值 ） 2个对象设置来完成 View 的加载。

如果 tryCreateView() 方法没有返回一个 View，那么就由系统生成 View。

### AsyncLayoutInflater

AsyncLayoutInflater 就是把 View 的一些初始化加载过程，放在子线程，结束后，在将结果回调到主线程。

```java
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    new AsyncLayoutInflater(this).inflate(R.layout.activity_main,null, new AsyncLayoutInflater.OnInflateFinishedListener(){
        @Override
        public void onInflateFinished(View view, int resid, ViewGroup parent) {
            setContentView(view);
            rv = findViewById(R.id.tv_right);
            rv.setLayoutManager(new V7LinearLayoutManager(MainActivity.this));
            rv.setAdapter(new RightRvAdapter(MainActivity.this));
        }
    });
}
```

AsyncLayoutInflater 的构造方法中做了三件事：

- 创建 BasicInflater
- 创建 Handler
- 创建 InflateThread

```java
public final class AsyncLayoutInflater {
    private static final String TAG = "AsyncLayoutInflater";

    LayoutInflater mInflater;
    Handler mHandler;
    InflateThread mInflateThread;

    public AsyncLayoutInflater(@NonNull Context context) {
        mInflater = new BasicInflater(context);
        mHandler = new Handler(mHandlerCallback);
        mInflateThread = InflateThread.getInstance();
    }

    @UiThread
    public void inflate(@LayoutRes int resid, @Nullable ViewGroup parent,
            @NonNull OnInflateFinishedListener callback) {
        if (callback == null) {
            throw new NullPointerException("callback argument may not be null!");
        }
        InflateRequest request = mInflateThread.obtainRequest();
        request.inflater = this;
        request.resid = resid;
        request.parent = parent;
        request.callback = callback;
        mInflateThread.enqueue(request);
    }
        ....

```

InflateThread 的作用是创建一个子线程，将 inflate 请求添加到阻塞队列中，并按顺序执行 BasicInflater.inflate 操作，不管 infalte 成功或失败后，都会将 request 消息发送给主线程做处理。

```java
private static class InflateThread extends Thread {
    private static final InflateThread sInstance;
    static {
        sInstance = new InflateThread();
        sInstance.start();
    }

    public static InflateThread getInstance() {
        return sInstance;
    }
        //生产者-消费者模型，阻塞队列
    private ArrayBlockingQueue<InflateRequest> mQueue = new ArrayBlockingQueue<>(10);
    //使用了对象池来缓存InflateThread对象，减少对象重复多次创建，避免内存抖动
    private SynchronizedPool<InflateRequest> mRequestPool = new SynchronizedPool<>(10);

    public void runInner() {
        InflateRequest request;
        try {
            //从队列中取出一条请求，如果没有则阻塞
            request = mQueue.take();
        } catch (InterruptedException ex) {
            // Odd, just continue
            Log.w(TAG, ex);
            return;
        }

        try {
            //inflate操作（通过调用BasicInflater类）
            request.view = request.inflater.mInflater.inflate(
                    request.resid, request.parent, false);
        } catch (RuntimeException ex) {
            // 回退机制：如果inflate失败，回到主线程去inflate
            Log.w(TAG, "Failed to inflate resource in the background! Retrying on the UI"
                    + " thread", ex);
        }
        //inflate成功或失败，都将request发送到主线程去处理
        Message.obtain(request.inflater.mHandler, 0, request)
                .sendToTarget();
    }

    @Override
    public void run() {
        //死循环（实际不会一直执行，内部是会阻塞等待的）
        while (true) {
            runInner();
        }
    }

    //从对象池缓存中取出一个InflateThread对象
    public InflateRequest obtainRequest() {
        InflateRequest obj = mRequestPool.acquire();
        if (obj == null) {
            obj = new InflateRequest();
        }
        return obj;
    }

    //对象池缓存中的对象的数据清空，便于对象复用
    public void releaseRequest(InflateRequest obj) {
        obj.callback = null;
        obj.inflater = null;
        obj.parent = null;
        obj.resid = 0;
        obj.view = null;
        mRequestPool.release(obj);
    }

    //将inflate请求添加到ArrayBlockingQueue（阻塞队列）中
    public void enqueue(InflateRequest request) {
        try {
            mQueue.put(request);
        } catch (InterruptedException e) {
            throw new RuntimeException(
                    "Failed to enqueue async inflate request", e);
        }
    }
}
```

mHandlerCallback 的作用是当子线程中 inflate 失败后，会继续再主线程中进行 inflate 操作，最终通过OnInflateFinishedListener 接口将 view 回调到主线程。

```java
private Callback mHandlerCallback = new Callback() {
    @Override
    public boolean handleMessage(Message msg) {
        InflateRequest request = (InflateRequest) msg.obj;
        if (request.view == null) {
            //view == null说明inflate失败
            //继续再主线程中进行inflate操作
            request.view = mInflater.inflate(
                    request.resid, request.parent, false);
        }
        //回调到主线程
        request.callback.onInflateFinished(
                request.view, request.resid, request.parent);
        mInflateThread.releaseRequest(request);
        return true;
    }
};
```

使用AsyncLayoutInflate主要有如下几个局限性：

1. 所有构建的 View 中不能直接使用 Handler 或者是调用 Looper.myLooper()，因为异步线程默认没有调用 Looper.prepare ()
2. 异步转换出来的 View 并没有被加到父 View 中，需要我们自己手动添加；
3. AsyncLayoutInflater 不支持设置 LayoutInflater.Factory 或者 LayoutInflater.Factory2
4. 同时缓存队列默认 10 的大小限制如果超过了10个则会导致主线程的等待
5. 使用单线程来做全部的 inflate 工作，如果一个界面中 layout 很多不一定能满足需求



[Android invalidate/postInvalidate/requestLayout-彻底厘清 - 简书 (jianshu.com)](https://www.jianshu.com/p/02073c90ef98)

[Android Window 如何确定大小 onMeasure()多次执行原因 - 简书 (jianshu.com)](https://www.jianshu.com/p/a7ab49462ebe)

[Android：6种高效 & 准确获取View坐标位置的方式 - 掘金 (juejin.cn)](https://juejin.cn/post/6844903975175602189)



