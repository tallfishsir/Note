# Android RecyclerView

## 基本使用

使用 RecyclerView 大概需要以下步骤：

- setLayoutManager(LayoutManager layout) 设置布局管理器
- addItemDecoration(ItemDecoration decor) 设置 ItemView 的分割线样式
- setItemAnimator(ItemAnimator animator) 设置 ItemView 的动画样式
- setAdapter(Adapter adapter) 设置 Adapter

### RecyclerView

#### 手势处理

RecyclerView 可以通过以下方法添加手势监听器，处理触摸和滑动事件：

- addOnItemTouchListener(OnItemTouchListener listener)：处理触摸事件
- addOnScrollListener(OnScrollListener listener)：处理滚动事件

RecyclerView.OnItemTouchListener 用于在 RecyclerView 中处理触摸事件，在实现该接口时，需要重写以下方法：

- onInterceptTouchEvent(RecyclerView rv, MotionEvent e)：用于在触摸事件被处理之前拦截它。返回 true 时，表示你已拦截并处理了触摸事件，此时不会触发 RecyclerView 的其他触摸事件处理方法。
- onTouchEvent(RecyclerView rv, MotionEvent e)：处理触摸事件。当 onInterceptTouchEvent() 返回 true 时，此方法将被调用。
- onRequestDisallowInterceptTouchEvent(boolean disallowIntercept)：在某些情况下（如嵌套滚动时），可以使用此方法告知父视图不要拦截子视图的触摸事件。

RecyclerView.OnScrollListener 用于在 RecyclerView 中监听滚动事件。可以通过实现此接口的方法来响应滚动事件，例如加载更多数据等。在实现该接口时，需要重写以下方法：

- onScrollStateChanged(RecyclerView recyclerView, int newState)：当 RecyclerView 的滚动状态发生变化时调用。newState 参数表示新的滚动状态。
  - SCROLL_STATE_IDLE：停止滑动时的状态
  - SCROLL_STATE_DRAGGING：手指拖动时的状态
  - SCROLL_STATE_SETTLING：惯性滑动时的状态
- onScrolled(RecyclerView recyclerView, int dx, int dy)：当 RecyclerView 发生滚动时调用。dx 和 dy 参数表示每帧内的滑动距离

#### 查找 View/ViewHolder

RecyclerView 查找某个 ViewItem 时，可以使用以下方法：

- findChildViewUnder(float x, float y)：查找位于指定坐标（x, y）下的 ItemView，x 和 y 是相对于 RecyclerView 的坐标。
- findContainingItemView(View view)：查找包含指定视图的 RecyclerView 的 ItemView。

RecyclerView 查找某个 ViewHolder 时，可以使用以下方法：

- findViewHolderForAdapterPosition(int position)：根据 Adapter 中的位置查找对应的 ViewHolder，如果该位置的 ViewHolder 当前未附加到屏幕上，此方法可能返回 null。
- findViewHolderForLayoutPosition(int position)：根据布局中的位置查找对应的 ViewHolder，此方法可能受到数据集更改的影响，在处理数据集更改时，最好使用 findViewHolderForAdapterPosition()。
- findContainingViewHolder(View view)：查找包含指定视图的 ViewHolder。

#### 自定义配置

RecyclerView 提供了一下一些方法来配置缓存和性能优化：

- setItemViewCacheSize(int size)：设置 RecyclerView 缓存 View 数量 (mCachedViews)， 默认大小为 2。
- setHasFixedSize(boolean hasFixedSize)：当 RecyclerView 的尺寸不受 Adapter 内容影响时，设置为 true 可以提高性能。
- setRecycledViewPool(RecycledViewPool pool)：设置一个自定义的 RecycledViewPool，在多个 RecyclerView 之间共享。

### LayoutManager

RecyclerView 可以通过设置不同的 LayoutManager 来实现不同的布局样式，常见的 LayoutManager 有：

- LinearLayoutManager： 此布局管理器将列表项排列为垂直或水平的线性列表
- GridLayoutManager: 此布局管理器将列表项排列为网格，可以指定要显示的列数
- StaggeredGridLayoutManager: 此布局管理器将列表项排列为瀑布流布局，每个项目可以具有不同的大小。可以指定列数或行数

#### 测量阶段

在测量阶段，LayoutManager 提供了两个方法，用于测量子 View 的尺寸：

- measureChild(View child, int widthUsed, int heightUsed) 
- measureChildWithMargins(View child, int widthUsed, int heightUsed)

测量阶段结束后，可以通过以下方法获取子 View 的测量结果：

- int getDecoratedMeasuredWidth(View child)：返回 child.getMeasuredWidth() 与 Decorations 叠加后的结果
- int getDecoratedMeasuredHeight(View child)：返回 child.getMeasuredHeight() 与 Decorations 叠加后的结果

#### 布局阶段

在子 View 布局阶段，LayoutManager 提供了两个方法，用于叠加了 Decorations 效果的布局子 View：

- layoutDecorated(View child, int left, int top, int right, int bottom)
- layoutDecoratedWithMargins(View child, int left, int top, int right, int bottom)

### RecyclerView.Adapter

Adapter 负责完成从数据到布局的转换过程，需要实现以下方法

- VH onCreateViewHolder(@NonNull ViewGroup parent, int viewType)：根据布局文件创建新的 ViewHolder
- onBindViewHolder(RecyclerView.ViewHolder, int)：绑定数据到已创建的 ViewHolder
- int getItemCount()：返回列表项的数量，确定需要显示多少个列表项
- int getItemViewType(int position)：返回位于 position位置的列表项对应的 ViewType 

#### 监听 ViewHolder 状态

RecyclerView.Adapter 通过以下方法监听 ViewHolder 状态：

- onViewRecycled(VH holder)：ViewHolder 确认被回收，要放进 RecyclerViewPool 中时
- onAttachedToWindow()：ViewHolder 移入屏幕时
- onViewDetachedFromWindow()：ViewHolder 移出屏幕时

#### 通知更新

Adapter 提供了以下方法用于通知数据集更新页面刷新：

- notifyDataSetChanged(): 通知 RecyclerView 数据集已更改，需要重新加载所有可见项，导致 RecyclerView 重新布局并重新绑定所有可见项，性能较低。
- notifyItemChanged(int position): 通知 RecyclerView 指定位置的数据已更改，需要重新绑定该项。传入的参数是数据集中已更改项的位置。
- notifyItemRangeChanged(int startPosition, int itemCount): 通知 RecyclerView 指定范围内的数据已更改，需要重新绑定这些项。传入的参数是数据集中已更改项的起始位置和更改项的数量。
- notifyItemInserted(int position): 通知 RecyclerView 在指定位置插入了一项。传入的参数是数据集中插入项的位置。
- notifyItemRangeInserted(int startPosition, int itemCount): 通知 RecyclerView 在指定位置范围内插入了多项。传入的参数是数据集中插入项的起始位置和插入项的数量。
- notifyItemRemoved(int position): 通知 RecyclerView 在指定位置移除了一项。传入的参数是数据集中移除项的位置。
- notifyItemRangeRemoved(int startPosition, int itemCount): 通知 RecyclerView 在指定位置范围内移除了多项。传入的参数是数据集中移除项的起始位置和移除项的数量。
- notifyItemMoved(int fromPosition, int toPosition): 通知 RecyclerView 一个项从一个位置移动到另一个位置。传入的参数是数据集中移动前和移动后的位置。

### RecyclerView.ViewHolder

ViewHolder 负责保存数据和 itemView 绑定关系。一般情况下，都会使用 ViewHolder.itemView 来设置点击事件监听，需要注意的是，在监听器内部要判断 position 不是无效列表位 RecyclerView.NO_POSITION 再响应点击事件。获取 ViewHolder 位置有以下两个方法：

- getAdapterPosition()：返回 ViewHolder 在 RecyclerView.Adapter 中的位置。

  这个位置是 Adapter 数据中的位置，可以用于从 Adapter  中获取或操作数据。但是，当 RecyclerView 处理动画或数据发生变化时，getAdapterPosition() 可能会返回 RecyclerView.NO_POSITION，表示 ViewHolder 当前没有有效的位置。因此，在使用此方法时，应始终检查其返回值是否有效。

- getLayoutPosition()：返回 ViewHolder 在 RecyclerView 布局中的位置。

  这个位置反映了当前布局中项目的位置，但可能不同于适配器中的位置。这是因为布局中的位置可能会受到添加、删除或移动动画的影响。在大多数情况下，应该使用 getAdapterPosition()，而不是 getLayoutPosition()，因为前者可以更准确地反映数据集的状态。

## LayoutManager

RecyclerView 的测量和布局阶段，主要依靠 LayoutManager 对象完成。

### onMeasure

测量阶段主要分为以下几个部分：

- LayoutManager 对象没有设置，调用 defaultOnMeasure() 计算尺寸，这时 RecyclerView 不显示任何数据
- LayoutManager 开启了自动测量功能，RecyclerView 会根据内容大小自动调整自身大小，不需要开发者手动设置宽度和高度，一般这种情况，可能会测量两次
- LayoutManager 没有开启自动测量功能，这种情况比较少

```java
protected void onMeasure(int widthSpec, int heightSpec) {
    if (mLayout == null) {
        // 第一种情况
        defaultOnMeasure(widthSpec, heightSpec);
        return;
    }
    if (mLayout.isAutoMeasureEnabled()) {
        // 第二种情况
    } else {
        // 第三种情况
    }
}
```

#### LayoutManager 为空

默认情况下的测量方法，会根据 RecyclerView 的 SpecMode 来获取不同的值。最后通过 setMeasuredDimension() 设置宽高。

```java
void defaultOnMeasure(int widthSpec, int heightSpec) {
    final int width = LayoutManager.chooseSize(widthSpec,
            getPaddingLeft() + getPaddingRight(),
            ViewCompat.getMinimumWidth(this));
    final int height = LayoutManager.chooseSize(heightSpec,
            getPaddingTop() + getPaddingBottom(),
            ViewCompat.getMinimumHeight(this));
    setMeasuredDimension(width, height);
}

public static int chooseSize(int spec, int desired, int min) {
    final int mode = View.MeasureSpec.getMode(spec);
    final int size = View.MeasureSpec.getSize(spec);
    switch (mode) {
        case View.MeasureSpec.EXACTLY:
            return size;
        case View.MeasureSpec.AT_MOST:
            return Math.min(size, Math.max(desired, min));
        case View.MeasureSpec.UNSPECIFIED:
        default:
            return Math.max(desired, min);
    }
}
```

#### LayoutManager 开启自动测量

如果 LayoutManager 启用自动测量功能，会执行以下逻辑：

- 调用 mLayout.onMeasure(mRecycler, mState, widthSpec, heightSpec)，让 LayoutManager 执行测量过程
- 如果宽度和高度的测量模式都是 MeasureSpec.EXACTLY 就结束测量过程
- 执行第一阶段布局：dispatchLayoutStep1()
- 设置测量规范并执行第二阶段布局：dispatchLayoutStep2()
- 根据子视图的尺寸设置测量结果：mLayout.setMeasuredDimensionFromChildren(widthSpec, heightSpec)
- 如果需要，执行第二次测量：mLayout.shouldMeasureTwice()

```java
if (mLayout.mAutoMeasure) {
    // 先获取测量模式
    final int widthMode = MeasureSpec.getMode(widthSpec);
    final int heightMode = MeasureSpec.getMode(heightSpec);
    
    final boolean skipMeasure = widthMode == MeasureSpec.EXACTLY && heightMode == MeasureSpec.EXACTLY;
    // 如果width和height都已经是精确值，就不再根据内容进行测量
    if (skipMeasure || mAdapter == null) {
        return;
    }
    if (mState.mLayoutStep == State.STEP_START) {
    	//布局第一步，主要进行一些初始化的工作
        dispatchLayoutStep1();
    }
    //将widthSpec和heightSpec传给LayoutManager对象
    mLayout.setMeasureSpecs(widthSpec, heightSpec);
    mState.mIsMeasuring = true;
    //布局第二步，真正进行children的测量和布局
    dispatchLayoutStep2();

    //根据第二步子View的情况决定自身的大小
    mLayout.setMeasuredDimensionFromChildren(widthSpec, heightSpec);

    //如果有父子尺寸属性互相依赖的情况，要改变参数重新进行一次
    if (mLayout.shouldMeasureTwice()) {
        mLayout.setMeasureSpecs(
                MeasureSpec.makeMeasureSpec(getMeasuredWidth(), MeasureSpec.EXACTLY),
                MeasureSpec.makeMeasureSpec(getMeasuredHeight(), MeasureSpec.EXACTLY));
        mState.mIsMeasuring = true;
        dispatchLayoutStep2();
        mLayout.setMeasuredDimensionFromChildren(widthSpec, heightSpec);
    }
}
```

#### LayoutManager 没有开启自动测量

如果 LayoutManager 没有启用自动测量功能，会执行以下逻辑：

- 如果设置了固定尺寸（mHasFixedSize=true），调用 mLayout.onMeasure(mRecycler, mState, widthSpec, heightSpec)，让 LayoutManager 执行测量过程
- 如果在测量过程中发生了适配器更新（mAdapterUpdateDuringMeasure=true），处理这些更新，并在必要时进行预布局
- 调用 mLayout.onMeasure(mRecycler, mState, widthSpec, heightSpec)，让 LayoutManager 执行测量过程

```java
if (mHasFixedSize) {
    mLayout.onMeasure(mRecycler, mState, widthSpec, heightSpec);
    return;
}
// custom onMeasure
if (mAdapterUpdateDuringMeasure) {
    eatRequestLayout();
    processAdapterUpdatesAndSetAnimationFlags();

    if (mState.mRunPredictiveAnimations) {
        mState.mInPreLayout = true;
    } else {
        mAdapterHelper.consumeUpdatesInOnePass();
        mState.mInPreLayout = false;
    }
    mAdapterUpdateDuringMeasure = false;
    resumeRequestLayout(false);
}

if (mAdapter != null) {
    mState.mItemCount = mAdapter.getItemCount();
} else {
    mState.mItemCount = 0;
}
eatRequestLayout();
mLayout.onMeasure(mRecycler, mState, widthSpec, heightSpec);
resumeRequestLayout(false);
mState.mInPreLayout = false;
```

### dispatchLayoutStep

在 RecyclerView 的测量布局过程中，State.mLayoutStep 记录了当前执行到哪一步骤，于此对应的是三个 dispatchLayoutStep 方法来完成 RecyclerView 测量布局工作。

| State.mLayoutStep | dispatchLayoutStep  | 说明                                                         |
| ----------------- | ------------------- | ------------------------------------------------------------ |
| STEP_START        | dispatchLayoutStep1 | STEP_START是 State.mLayoutStep 默认值，执行完dispatchLayoutStep1()后会将该状态设置 STEP_LAYOUT |
| STEP_LAYOUT       | dispatchLayoutStep2 | 表明 RecyclerView 处于布局阶段，执行完dispatchLayoutStep2()后会将该状态设置 STEP_ANIMATIONS |
| STEP_ANIMATIONS   | dispatchLayoutStep3 | 表明 RecyclerView 处于执行动画阶段，执行完dispatchLayoutStep3()后会将该状态重新设置为 STEP_START |

#### dispatchLayoutStep1

在 dispatchLayoutStep1() 中，主要处理了预布局阶段，包括处理适配器更新、计算动画的开始值，以及为预测性动画做准备。整个过程涉及多个步骤，例如遍历子视图、记录预布局信息、执行 LayoutManager 的预布局等。这个阶段为接下来的布局过程和动画计算奠定了基础。

- 填充剩余的滚动值并设置 mLayoutStep=State.STEP_START
- 拦截请求布局，以防止在此过程中发生意外的布局
- 清除视图信息存储以准备新的布局过程
- 如果需要运行简单动画mRunSimpleAnimations=true，记录所有子视图它们在预布局阶段的状态
- 如果需要运行预测性动画 mRunPredictiveAnimations=true
  - 保存旧位置，以便 LayoutManager 可以执行映射逻辑
  - 执行预布局，让 LayoutManager 使用旧位置布局所有子视图
  - 遍历所有子视图并记录它们在预布局阶段的状态
- 停止拦截请求布局
- 准备进入第二阶段设置 mLayoutStep=State.STEP_LAYOUT

```java
private void dispatchLayoutStep1() {
    mState.assertLayoutStep(State.STEP_START);
    fillRemainingScrollValues(mState);
    mState.mIsMeasuring = false;
    startInterceptRequestLayout();
    mViewInfoStore.clear();
    onEnterLayoutOrScroll();
    processAdapterUpdatesAndSetAnimationFlags();
    saveFocusInfo();
    mState.mTrackOldChangeHolders = mState.mRunSimpleAnimations && mItemsChange
    mItemsAddedOrRemoved = mItemsChanged = false;
    mState.mInPreLayout = mState.mRunPredictiveAnimations;
    mState.mItemCount = mAdapter.getItemCount();
    findMinMaxChildLayoutPositions(mMinMaxLayoutPositions);
    if (mState.mRunSimpleAnimations) {
        // Step 0: Find out where all non-removed items are, pre-layout
        int count = mChildHelper.getChildCount();
        for (int i = 0; i < count; ++i) {
            final ViewHolder holder = getChildViewHolderInt(mChildHelper.getChi
            if (holder.shouldIgnore() || (holder.isInvalid() && !mAdapter.hasSt
                continue;
            }
            final ItemHolderInfo animationInfo = mItemAnimator
                    .recordPreLayoutInformation(mState, holder,
                            ItemAnimator.buildAdapterChangeFlagsForAnimations(h
                            holder.getUnmodifiedPayloads());
            mViewInfoStore.addToPreLayout(holder, animationInfo);
            if (mState.mTrackOldChangeHolders && holder.isUpdated() && !holder.
                    && !holder.shouldIgnore() && !holder.isInvalid()) {
                long key = getChangedHolderKey(holder);
                mViewInfoStore.addToOldChangeHolders(key, holder);
            }
        }
    }
    if (mState.mRunPredictiveAnimations) {
        //预布局
        saveOldPositions();
        final boolean didStructureChange = mState.mStructureChanged;
        mState.mStructureChanged = false;
        mLayout.onLayoutChildren(mRecycler, mState);
        mState.mStructureChanged = didStructureChange;
        for (int i = 0; i < mChildHelper.getChildCount(); ++i) {
            final View child = mChildHelper.getChildAt(i);
            final ViewHolder viewHolder = getChildViewHolderInt(child);
            if (viewHolder.shouldIgnore()) {
                continue;
            }
            if (!mViewInfoStore.isInPreLayout(viewHolder)) {
                int flags = ItemAnimator.buildAdapterChangeFlagsForAnimations(v
                boolean wasHidden = viewHolder
                        .hasAnyOfTheFlags(ViewHolder.FLAG_BOUNCED_FROM_HIDDEN_L
                if (!wasHidden) {
                    flags |= ItemAnimator.FLAG_APPEARED_IN_PRE_LAYOUT;
                }
                final ItemHolderInfo animationInfo = mItemAnimator.recordPreLay
                        mState, viewHolder, flags, viewHolder.getUnmodifiedPayl
                if (wasHidden) {
                    recordAnimationInfoIfBouncedHiddenView(viewHolder, animatio
                } else {
                    mViewInfoStore.addToAppearedInPreLayoutHolders(viewHolder, 
                }
            }
        }
        clearOldPositions();
    } else {
        clearOldPositions();
    }
    onExitLayoutOrScroll();
    stopInterceptRequestLayout(false);
    mState.mLayoutStep = State.STEP_LAYOUT;
}
```

#### dispatchLayoutStep2

在 dispatchLayoutStep2() 方法中，主要执行了实际布局并处理了动画。它调用了 LayoutManager 的 onLayoutChildren() 方法进行布局，并根据布局的结果更新了状态。同时，它还处理了一些与动画相关的设置。当此方法执行完成后，子 View 的宽高信息都可以获取。

- 开始拦截请求布局，以防止在此过程中发生意外的布局
- 检查 mState 状态正常
- 使用 mAdapterHelper.consumeUpdatesInOnePass() 一次性处理所有适配器更新
- 更新 mState 中的 item 数量
- 如果有挂起的保存状态（如配置更改或屏幕旋转），尝试恢复布局状态
- 设置 mState.mInPreLayout 为 false，表示当前不处于预布局阶段
- 调用 LayoutManager 的 onLayoutChildren() 方法进行实际的布局
- 如果存在 ItemAnimator，则 mState.mRunSimpleAnimations 为 true
- 将布局步骤设置为 State.STEP_ANIMATIONS，表示下一步是动画处理
- 停止拦截请求布局

```java
private void dispatchLayoutStep2() {
    startInterceptRequestLayout();
    onEnterLayoutOrScroll();
    mState.assertLayoutStep(State.STEP_LAYOUT | State.STEP_ANIMATIONS);
    mAdapterHelper.consumeUpdatesInOnePass();
    mState.mItemCount = mAdapter.getItemCount();
    mState.mDeletedInvisibleItemCountSincePreviousLayout = 0;
    if (mPendingSavedState != null && mAdapter.canRestoreState()) {
        if (mPendingSavedState.mLayoutState != null) {
            mLayout.onRestoreInstanceState(mPendingSavedState.mLayoutState);
        }
        mPendingSavedState = null;
    }
    // Step 2: Run layout
    mState.mInPreLayout = false;
    mLayout.onLayoutChildren(mRecycler, mState);
    mState.mStructureChanged = false;
    mState.mRunSimpleAnimations = mState.mRunSimpleAnimations && mItemAnimator !=
    mState.mLayoutStep = State.STEP_ANIMATIONS;
    onExitLayoutOrScroll();
    stopInterceptRequestLayout(false);
}
```

#### dispatchLayoutStep3

dispatchLayoutStep3 会执行在 dispatchLayoutStep1 中保存的动画信息。最后将 mLayoutStep 设置为STEP_START 状态。来保证第二次 layout 时仍会执行上面的流程。

```java
private void dispatchLayoutStep3() {
    ...
    //mLayoutStep设置为STEP_ANIMATIONS状态
    mState.mLayoutStep = State.STEP_START;
}
```

### onLayout

布局阶段最重要的逻辑都在 dispatchLayout 方法中，根据不同的状态，执行不同的逻辑，来保证 RecyclerView 一定会经历 dispatchLayoutStep1、dispatchLayoutStep2、dispatchLayoutStep3 三个流程。

```java
@Override
protected void onLayout(boolean changed, int l, int t, int r, int b) {
    TraceCompat.beginSection(TRACE_ON_LAYOUT_TAG);
    dispatchLayout();
    TraceCompat.endSection();
    mFirstLayoutComplete = true;
}

void dispatchLayout() {
    //mAdapter为空，不加载显示数据
    if (mAdapter == null) {
        Log.e(TAG, "No adapter attached; skipping layout");
        return;
    }
    //mLayout为空，不加载显示数据
    if (mLayout == null) {
        Log.e(TAG, "No layout manager attached; skipping layout");
        return;
    }
    mState.mIsMeasuring = false;
    if (mState.mLayoutStep == State.STEP_START) {
        dispatchLayoutStep1();
        mLayout.setExactMeasureSpecsFrom(this);
        dispatchLayoutStep2();
    } else if (mAdapterHelper.hasUpdates()
               || mLayout.getWidth() != getWidth()
               || mLayout.getHeight() != getHeight()) {
        mLayout.setExactMeasureSpecsFrom(this);
        dispatchLayoutStep2();
    } else {
        // always make sure we sync them (to ensure mode is exact)
        mLayout.setExactMeasureSpecsFrom(this);
    }
    dispatchLayoutStep3();
}
```

### LinearLayoutManager

dispatchLayoutStep2 中说明，RecyclerView 的子 View 的布局都是由 LayoutManager.onLayoutChildren 方法实现，以 LinearLayoutManager 为例，整理流程是：

- 确定子 View 的布局方向，再确定锚点信息，找到锚点坐标（mCoordinate）和锚点 View 索引（mPosition）
- 回收 RecyclerView 的子 View
- 根据锚点信息，调用 fill() 方法，按照不同的方向重新填充子 View

```java
public void onLayoutChildren(RecyclerView.Recycler recycler, RecyclerView.State state) {
    ...
    //确定布局方向
    resolveShouldLayoutReverse();
    ...
    if (!mAnchorInfo.mValid || mPendingScrollPosition != NO_POSITION ||
            mPendingSavedState != null) {
        //重置锚点信息
        mAnchorInfo.reset();
        //是否从end开始进行布局。因为mShouldReverseLayout和mStackFromEnd默认都是false，那么我们这里可以考虑按照默认的情况来进行分析，也就是mLayoutFromEnd也是false
        mAnchorInfo.mLayoutFromEnd = mShouldReverseLayout ^ mStackFromEnd;
        //确定锚点信息，位置和坐标
        updateAnchorInfoForLayout(recycler, state, mAnchorInfo);
        //设置锚点有效
        mAnchorInfo.mValid = true;
    }
    ...
    onAnchorReady(recycler, state, mAnchorInfo, firstLayoutDirection);
    //暂时回收子View
    detachAndScrapAttachedViews(recycler);
    mLayoutState.mInfinite = resolveIsInfinite();
    mLayoutState.mIsPreLayout = state.isPreLayout();
    
    //根据锚点信息，填充子View
    if (mAnchorInfo.mLayoutFromEnd) {
        //根据mAnchorInfo初始化LayoutState相关信息
        updateLayoutStateToFillStart(mAnchorInfo);
        mLayoutState.mExtra = extraForStart;
        //填充子View
        fill(recycler, mLayoutState, state, false);
        ...
        //根据mAnchorInfo初始化LayoutState相关信息
        updateLayoutStateToFillEnd(mAnchorInfo);
        mLayoutState.mExtra = extraForEnd;
        mLayoutState.mCurrentPosition += mLayoutState.mItemDirection;
        //填充子View
        fill(recycler, mLayoutState, state, false);
        ...
    } else {
        //根据mAnchorInfo初始化LayoutState相关信息
        updateLayoutStateToFillEnd(mAnchorInfo);
        mLayoutState.mExtra = extraForEnd;
        //填充子View
        fill(recycler, mLayoutState, state, false);
        ...
        //根据mAnchorInfo初始化LayoutState相关信息
        updateLayoutStateToFillStart(mAnchorInfo);
        mLayoutState.mExtra = extraForStart;
        mLayoutState.mCurrentPosition += mLayoutState.mItemDirection;
        //填充子View
        fill(recycler, mLayoutState, state, false);
        ...
    }
    ...
}
```

#### 锚点信息 AnchorInfo

AnchorInfo 用于描述具体的位置信息

```java
static class AnchorInfo {
    //锚点参考View在整个数据中的position信息，即它是第几个View
    int mPosition;
    //锚点的具体坐标信息，填充子View的起始坐标
    int mCoordinate;
    //是否从底部开始布局
    boolean mLayoutFromEnd;
    //默认是false，一次测量后，设置true。onLayout完成后，会调用reset方法重置为false
    boolean mValid;    
}
```

#### 确定布局方向和锚点信息

resolveShouldLayoutReverse 确定了布局方向后，会调用 updateAnchorInfoForLayout 来计算锚点信息，完成后，就可以得到布局时所需要的第一个 view 的索引和位置坐标。

```java
private void resolveShouldLayoutReverse() {
    if (mOrientation == VERTICAL || !isLayoutRTL()) {
        mShouldReverseLayout = mReverseLayout;
    } else {
        mShouldReverseLayout = !mReverseLayout;
    }
}

//是否从end开始进行布局。因为mShouldReverseLayout和mStackFromEnd默认都是false，那么我们这里可以考虑按照默认的情况来进行分析，也就是mLayoutFromEnd也是false
mAnchorInfo.mLayoutFromEnd = mShouldReverseLayout ^ mStackFromEnd;

private void updateAnchorInfoForLayout(RecyclerView.Recycler recycler, RecyclerView.State state, AnchorInfo anchorInfo) {
    // 第一种计算方法
    if (updateAnchorFromPendingData(state, anchorInfo)) {
        return;
    }
    
    // 第二种计算方法
    if (updateAnchorFromChildren(recycler, state, anchorInfo)) {
        return;
    }
    
    // 第三种计算方法
    anchorInfo.assignCoordinateFromPadding();
    anchorInfo.mPosition = mStackFromEnd ? state.getItemCount() - 1 : 0;
}
```

第一种计算方式，一般有两种情况出现：

- RecyclerView 被重建，期间回调了 onSaveInstanceState 方法，目的是恢复上次的布局
- RecyclerView 调用了 scrollToPosition 之类的犯法，目的是让 RecyclerView 滚动到准确的位置上

第二种计算方式，是从子 View 上来计算锚点，一般有两种情况出现：

- 当前含有拥有焦点的子 View，根据此 View 的位置来计算锚点
- 当前不含有拥有焦点的子 View，根据布局方向（mLayoutFromEnd）获取可见的第一个或者最后一个 ItemView

```java
private boolean updateAnchorFromChildren(RecyclerView.Recycler recycler, RecyclerView.State state, AnchorInfo anchorInfo) {
	//没有数据，返回false，表示更新失败
	if (getChildCount() == 0) {
		return false;
	}
	final View focused = getFocusedChild();
	//优先选取获得焦点的子View作为锚点
	if (focused != null && anchorInfo.isViewValidAsAnchor(focused, state)) {
		//计算获取焦点的子view的位置信息
		anchorInfo.assignFromViewAndKeepVisibleRect(focused);
		return true;
	}
	if (mLastStackFromEnd != mStackFromEnd) {
		return false;
	}
	//根据锚点的设置信息，从底部或者顶部获取子View信息
	View referenceChild = anchorInfo.mLayoutFromEnd ? findReferenceChildClosestToEnd(recycler, state) : findReferenceChildClosestToStart(recycler, state);
	if (referenceChild != null) {
		anchorInfo.assignFromView(referenceChild);
		...
		return true;
	}
	return false;
}

public void assignFromView(View child) {
    if (mLayoutFromEnd) {
        //如果是从底部布局，那么获取child的底部的位置设置为锚点
        mCoordinate = mOrientationHelper.getDecoratedEnd(child) + mOrientationHelper.getTotalSpaceChange();
    } else {
        //如果是从顶部开始布局，那么获取child的顶部的位置设置为锚点坐标(这里要考虑ItemDecorator的情况)
        mCoordinate = mOrientationHelper.getDecoratedStart(child);
    }
    //mPosition赋值为参考View的position
    mPosition = getPosition(child);
}
```

第三种计算方式，是前面两个方法计算失败，所采用的默认的计算方式，比如第一次加载数据时。默认情况下，mCoordinate 取值是 RecyclerView 设置的 StartPadding 数值，mPosition 是 0。

```java
void assignCoordinateFromPadding() {
    mCoordinate = mLayoutFromEnd
            ? mOrientationHelper.getEndAfterPadding()
            : mOrientationHelper.getStartAfterPadding();
}
```

#### 回收子 View

detachAndScrapAttachedViews() 会对所有 ItemView 进行回收，放入缓存，后续再通过 recycler 重新取出布局。

#### 填充子 View

根据 mAnchorInfo.mLayoutFromEnd 来判断填充方向，然后通过 updateLayoutStateToFillStart()/updateLayoutStateToFillEnd() 设置填充时所需要的位置等信息。LayoutState 用于记录当前的加载状态，比如当前绘制的偏移量，RecyclerView 还剩余多少空间等信息。

```java
static class LayoutState {
    //布局位置的偏移量(初始值应该是锚点里面设置的mCoordinate)
    int mOffset;
    //布局方向上剩余的空间大小
    int mAvailable;
    //当前布局的position
    int mCurrentPosition;
    //数据取值顺序方向
    int mItemDirection;
    //填充view顺序方向
    int mLayoutDirection;
    //不创建新视图的情况下进行滚动的距离，比如某个View只显示了上半部分，这时候往上滑动一半距离的话以内，是不需要创建新的子View的。这个一般距离值就是mScrollingOffset
	int mScrollingOffset;
}
```

最后至少调用两次 fill() 进行不同方向的填充，每一次添加 ItemView 后，都通过 LayoutChunkResult 类来记录加载子 View 结果情况，比如已经填充的数量等信息。

```java
int fill(RecyclerView.Recycler recycler, LayoutState layoutState,
        RecyclerView.State state, boolean stopOnFocusable) {
    //可用空间大小
    final int start = layoutState.mAvailable;
    //剩余绘制空间=可用区域+扩展空间。
    int remainingSpace = layoutState.mAvailable + layoutState.mExtra;
    LayoutChunkResult layoutChunkResult = mLayoutChunkResult;
    while ((layoutState.mInfinite || remainingSpace > 0) && layoutState.hasMore(state)) {
        //初始化layoutChunkResult
        layoutChunkResult.resetInternal();
        //实际添加ItemView，然后将绘制的相关信息保存到layoutChunkResult
        layoutChunk(recycler, state, layoutState, layoutChunkResult);
        ...
        //根据所添加的child消费的高度更新layoutState的偏移量
        layoutState.mOffset += layoutChunkResult.mConsumed * layoutState.mLayoutDirection;

        if (!layoutChunkResult.mIgnoreConsumed || mLayoutState.mScrapList != null
                || !state.isPreLayout()) {
            //计算剩余可用空间
            layoutState.mAvailable -= layoutChunkResult.mConsumed;
            remainingSpace -= layoutChunkResult.mConsumed;
        }
        ...
    }
    ...
    return start - layoutState.mAvailable;
}

void layoutChunk(RecyclerView.Recycler recycler, RecyclerView.State state,
        LayoutState layoutState, LayoutChunkResult result) {
    //通过缓存获取当前position所需要展示的ViewHolder的View
    View view = layoutState.next(recycler);
    ...
    RecyclerView.LayoutParams params = (RecyclerView.LayoutParams) view.getLayoutParams();
    if (layoutState.mScrapList == null) {
        //填充子View
        if (mShouldReverseLayout == (layoutState.mLayoutDirection
                == LayoutState.LAYOUT_START)) {
            addView(view);
        } else {
            addView(view, 0);
        }
    }
    ...
    //调用measure测量view。这里会考虑到父类的padding
    measureChildWithMargins(view, 0, 0);
    //将本次子View消费的区域设置为子view的高(或者宽)
    result.mConsumed = mOrientationHelper.getDecoratedMeasurement(view);
    ...
    //调用child.layout方法进行布局(这里会考虑到view的ItemDecorator等信息)
    layoutDecoratedWithMargins(view, left, top, right, bottom);
}
```

实际上完成子 View 填充的逻辑是在 layoutChunk() 中处理，核心流程是：

- LayoutState.next() 从缓存中获取子 View
- 通过 addView() 将子 View 添加到布局中
- 调用 measureChildWithMargins() 测量子 View，计算宽高
- 调用 layoutDecoratedWithMargins() 布局子 View

## Recycler

RecyclerView 的复用机制由 RecyclerView.Recycler 负责，它用来完成 View 的回收和复用工作。Recycler 依次通过 Scrap、CacheView、ViewCacheExtension、RecycledViewPool 四级缓存实现。以下是一些主要的缓存获取方法：

- getViewForPosition(int position)

  根据给定的位置从适配器中获取一个视图。这个方法首先会从已回收的视图池中查找，如果找到合适的视图，则直接从池中获取并重新绑定数据。如果没有找到合适的视图，它将创建一个新的视图并绑定数据。

- getViewForPosition(int position, boolean dryRun)

  额外的参数 dryRun。如果设置为 true，则在需要创建新视图时不会实际创建和绑定视图。这在预布局阶段有用，因为它允许布局管理器预测将要创建的视图数量而不实际创建它们。

- getScrapViewForPosition(int position, int type, boolean dryRun):

  这个方法尝试从废弃视图列表中获取给定位置和类型的视图。如果 dryRun 为 true，则即使需要创建新视图，也不会实际创建和绑定视图。这在预布局阶段有用。

- getRecycledViewPool():

  获取与 RecyclerView 关联的 RecycledViewPool。这个池用于在回收和重用视图时存储视图。

### 缓存机制

#### 一级缓存 Scrap

这是优先级最高的缓存，会先到这一分为 mAttachedScrap 和 mChangedScrap，RecyclerView 在获取 ViewHolder 时，优先会到这两个缓存获取。它不参与滑动时的回收复用，这是作为重新布局时的一种临时缓存。它的目的是，缓存屏幕分离出来的 ViewHolder，但又即将添加到屏幕上的 ViewHolder，以此省去不必要的重新加载与绑定工作。

mAttachedScrap 中存储的是屏幕中原封不动的ViewHolder，具体的使用场景是从屏幕上分离出来的 ViewHolder，但又即将添加到屏幕上的 ViewHolder。

mChangedScrap 中存储的是位置会发生移动的ViewHolder，注意只是位置发生移动，内容仍旧是原封不动的。

```java
public final class Recycler {
    final ArrayList<ViewHolder> mAttachedScrap = new ArrayList<>();
    ArrayList<ViewHolder> mChangedScrap = null;
    
    void scrapView(View view) {
        final ViewHolder holder = getChildViewHolderInt(view);
        if (holder.hasAnyOfTheFlags(ViewHolder.FLAG_REMOVED | ViewHolder.FLAG_INVALID)
                || !holder.isUpdated() || canReuseUpdatedViewHolder(holder)) {
            if (holder.isInvalid() && !holder.isRemoved() && !mAdapter.hasStableIds()) {
                throw new IllegalArgumentException("Called scrap view with an invalid view."
                        + " Invalid views cannot be reused from scrap, they should rebound from recycler pool." + exceptionLabel());
            }
            holder.setScrapContainer(this, false);
            mAttachedScrap.add(holder);
        } else {
            if (mChangedScrap == null) {
                mChangedScrap = new ArrayList<ViewHolder>();
            }
            holder.setScrapContainer(this, true);
            mChangedScrap.add(holder);
        }
    }
}
```

#### 二级缓存 CacheView

CacheView 是一个以 ViewHolder 为单位，负责在 RecyclerView 列表内容位置发生变化时，对刚刚移出移入屏幕的  View 进行回收复用的缓存。在复用的时候，不需要重新创建和绑定，内容不发生改变。

CacheView 的最大默认缓存个数是2，通常大小是3，由默认大小2+预取个数1相加。如果超过最大缓存数量，会先调用 recycleCachedViewAt(0)，把最先缓存进来的 ViewHolder 回收进 RecycledViewPool 中。

```java
public final class Recycler {
    final ArrayList<ViewHolder> mCachedViews = new ArrayList<ViewHolder>();
    static final int DEFAULT_CACHE_SIZE = 2;
    int mViewCacheMax = DEFAULT_CACHE_SIZE;
    
    void recycleViewHolderInternal(ViewHolder holder) {
        ...
        if (forceRecycle || holder.isRecyclable()) {
            if (mViewCacheMax > 0
                    && !holder.hasAnyOfTheFlags(ViewHolder.FLAG_INVALID
                    | ViewHolder.FLAG_REMOVED
                    | ViewHolder.FLAG_UPDATE
                    | ViewHolder.FLAG_ADAPTER_POSITION_UNKNOWN)) {
                // Retire oldest cached view
                int cachedViewSize = mCachedViews.size();
                if (cachedViewSize >= mViewCacheMax && cachedViewSize > 0) {
                    recycleCachedViewAt(0);
                    cachedViewSize--;
                }
                mCachedViews.add(targetCacheIndex, holder);
                cached = true;
            }
            ...
        } 
    	...
    }
}
```

#### 三级缓存 ViewCacheExtension

自定义缓存

#### 四级缓存 RecycledViewPool

RecycledViewPool 是 SparseArray 以 ViewHolder 的 ViewType 作为区分来嵌套 ArrayList 进行存储的。RecycledViewPool 在进行回收时，只是回收一个该 ViewType 的 ViewHolder 对象，并没有保存下原来的 ViewHolder 内容。在复用时会调用 onBindViewHolder() 重新进行绑定。

```java
public static class RecycledViewPool {
    static class ScrapData {
        final ArrayList<ViewHolder> mScrapHeap = new ArrayList<>();
        int mMaxScrap = DEFAULT_MAX_SCRAP;
        long mCreateRunningAverageNs = 0;
        long mBindRunningAverageNs = 0;
    }
    SparseArray<ScrapData> mScrap = new SparseArray<>();
}
```

### 复用 ViewHolder

RecyclerView 对 ViewHolder 的复用是在 layoutChunk 中 LayoutState.next() 方法开始。通过调用 RecyclerView.getViewForPosition() 来根据位置获取 ViewHolder。

```java
View next(RecyclerView.Recycler recycler) {
    if (mScrapList != null) {
        return nextViewFromScrapList();
    }
    final View view = recycler.getViewForPosition(mCurrentPosition);
    mCurrentPosition += mItemDirection;
    return view;
}

public View getViewForPosition(int position) {
    return getViewForPosition(position, false);
}

View getViewForPosition(int position, boolean dryRun) {
    return tryGetViewHolderForPositionByDeadline(position, dryRun, FOREVER_NS).itemView;
}
```

#### 通过 Position 方式获取 ViewHolder

通过 Position 方式获取通常情况下，都是某一个 ItemView 对应的 ViewHolder 被更新导致的。所以屏幕上其他的 ViewHolder 可以快速对应原来的 ItemView

```java
ViewHolder tryGetViewHolderForPositionByDeadline(int position,
        boolean dryRun, long deadlineNs) {
    ViewHolder holder = null;
    //预布局阶段，从mChangedScrap获取
    if (mState.isPreLayout()) {
        holder = getChangedScrapViewForPosition(position);
        fromScrapOrHiddenOrCache = holder != null;
    }
    
    if (holder == null) {
        //通过position，分别从mAttachedScrap、mCacheView获取ViewHolder
        holder = getScrapOrHiddenOrCachedHolderForPosition(position, dryRun);
        if (holder != null) {
            //如果获取的holder无效，做清理操作，然后重新放入缓存中（CacheView和RecycledViewPool）
            if (!validateViewHolderForOffsetPosition(holder)) {
                if (!dryRun) {
                    holder.addFlags(ViewHolder.FLAG_INVALID);
                    if (holder.isScrap()) {
                        removeDetachedView(holder.itemView, false);
                        holder.unScrap();
                    } else if (holder.wasReturnedFromScrap()) {
                        holder.clearReturnedFromScrapFlag();
                    }
                    recycleViewHolderInternal(holder);
                }
                holder = null;
            } else {
                fromScrapOrHiddenOrCache = true;
            }
        }
    }
}
```

#### 通过 ViewType 方式获取 ViewHolder

通过 ViewType 方式获取 ViewHolder 会依次从 mAttachedScrap、CacheView、RecycledViewPool 中获取，如果都没有，就通过 createViewHolder 创建一个新的 ViewHolder，最后通过 tryBindViewHolderByDeadline 绑定数据。

```java
ViewHolder tryGetViewHolderForPositionByDeadline(int position,
        boolean dryRun, long deadlineNs) {
    ...
    if (holder == null) {
        final int offsetPosition = mAdapterHelper.findPositionOffset(position);
        if (offsetPosition < 0 || offsetPosition >= mAdapter.getItemCount()) {
            throw new IndexOutOfBoundsException("Inconsistency detected. Invalid item "
                    + "position " + position + "(offset:" + offsetPosition + ")."
                    + "state:" + mState.getItemCount() + exceptionLabel());
        }
        final int type = mAdapter.getItemViewType(offsetPosition);
        //通过id来寻找ViewHolder
        if (mAdapter.hasStableIds()) {
            holder = getScrapOrCachedViewForId(mAdapter.getItemId(offsetPosition),
                    type, dryRun);
            if (holder != null) {
                holder.mPosition = offsetPosition;
                fromScrapOrHiddenOrCache = true;
            }
        }
        if (holder == null && mViewCacheExtension != null) {
            //从三级缓存ViewCacheExtension获取ViewHolder
            final View view = mViewCacheExtension
                    .getViewForPositionAndType(this, position, type);
            if (view != null) {
                holder = getChildViewHolder(view);
            }
        }
        if (holder == null) {
            //从四级缓存RecycledViewPool获取ViewHolder
            holder = getRecycledViewPool().getRecycledView(type);
            if (holder != null) {
                holder.resetInternal();
                if (FORCE_INVALIDATE_DISPLAY_LIST) {
                    invalidateDisplayListInt(holder);
                }
            }
        }
        if (holder == null) {
            long start = getNanoTime();
            //调用Adapter的onCreateViewHolder方法创建一个新的ViewHolder
            holder = mAdapter.createViewHolder(RecyclerView.this, type);
        }
    }
    ...
    boolean bound = false;
    if (mState.isPreLayout() && holder.isBound()) {
        holder.mPreLayoutPosition = position;
    } else if (!holder.isBound() || holder.needsUpdate() || holder.isInvalid()) {
        //bindViewHolder绑定ViewHolder
        final int offsetPosition = mAdapterHelper.findPositionOffset(position);
        bound = tryBindViewHolderByDeadline(holder, offsetPosition, position, deadlineNs);
    }
}
```

### 回收 ViewHolder

#### 回收到 Scrap

回收 Scrap 的逻辑一般都是由 detachAndScrapAttachedViews() 触发，实际回收发生在 Recycler.scrapView() 中。一般有两种情况会出现：

- 手动调用的 LayoutManager 相关方法
- RecyclerView 进行了一次布局（调用了 requestLayout 方法）

```java
public void detachAndScrapAttachedViews(@NonNull Recycler recycler) {
    final int childCount = getChildCount();
    for (int i = childCount - 1; i >= 0; i--) {
        final View v = getChildAt(i);
        scrapOrRecycleView(recycler, i, v);
    }
}

private void scrapOrRecycleView(Recycler recycler, int index, View view) {
    final ViewHolder viewHolder = getChildViewHolderInt(view);
    if (viewHolder.shouldIgnore()) {
        if (DEBUG) {
            Log.d(TAG, "ignoring view " + viewHolder);
        }
        return;
    }
    if (viewHolder.isInvalid() && !viewHolder.isRemoved()
            && !mRecyclerView.mAdapter.hasStableIds()) {
        removeViewAt(index);
        recycler.recycleViewHolderInternal(viewHolder);
    } else {
        detachViewAt(index);
        recycler.scrapView(view);
        mRecyclerView.mViewInfoStore.onViewDetached(viewHolder);
    }
}
```

#### 回收到 CacheView

CacheView 回收主要在于 Recycler.recycleViewHolderInternal() 中，一般有三种情况会出现：

- 重新布局中回收，比如调用了 Adapter.notifyDataSetChange
- 复用时，一级缓存获取到的 ViewHolder 相关数据已经失效（比如 Position），会从一级缓存中移除
- 当调用 removeAnimatingView()，如果当前 ViewHolder 已经被标记为 REMOVE，会回收到二级缓存

```java
void recycleViewHolderInternal(ViewHolder holder) {
    ...
    boolean cached = false;
    boolean recycled = false;
    if (forceRecycle || holder.isRecyclable()) {
        if (mViewCacheMax > 0
                && !holder.hasAnyOfTheFlags(ViewHolder.FLAG_INVALID
                | ViewHolder.FLAG_REMOVED
                | ViewHolder.FLAG_UPDATE
                | ViewHolder.FLAG_ADAPTER_POSITION_UNKNOWN)) {
            int cachedViewSize = mCachedViews.size();
            if (cachedViewSize >= mViewCacheMax && cachedViewSize > 0) {
                recycleCachedViewAt(0);
                cachedViewSize--;
            }
            mCachedViews.add(targetCacheIndex, holder);
            cached = true;
        }
        if (!cached) {
            addViewHolderToRecycledViewPool(holder, true);
            recycled = true;
        }
    } 
	...
}
```

#### 回收到 RecycledViewPool

CacheView 回收主要在于 Recycler.addViewHolderToRecycledViewPool() 中，一般是 CacheView 无法回收时使用。

```java
void addViewHolderToRecycledViewPool(@NonNull ViewHolder holder, boolean dispatchRecycled) {
    clearNestedRecyclerViewIfNotNested(holder);
    if (holder.hasAnyOfTheFlags(ViewHolder.FLAG_SET_A11Y_ITEM_DELEGATE)) {
        holder.setFlags(0, ViewHolder.FLAG_SET_A11Y_ITEM_DELEGATE);
        ViewCompat.setAccessibilityDelegate(holder.itemView, null);
    }
    if (dispatchRecycled) {
        dispatchViewRecycled(holder);
    }
    holder.mOwnerRecyclerView = null;
    getRecycledViewPool().putRecycledView(holder);
}

public void putRecycledView(ViewHolder scrap) {
    final int viewType = scrap.getItemViewType();
    final ArrayList<ViewHolder> scrapHeap = getScrapDataForType(viewType).mScrapHeap;
    if (mScrap.get(viewType).mMaxScrap <= scrapHeap.size()) {
        return;
    }
    if (DEBUG && scrapHeap.contains(scrap)) {
        throw new IllegalArgumentException("this scrap item already exists");
    }
    scrap.resetInternal();
    scrapHeap.add(scrap);
}
```

## ItemDecoration

ItemDecoration 用于给每个 ItemView 增加装饰效果，一般用来绘制分隔线。自定义 ItemDecoration 需要继承 RecyclerView.ItemDecoration，重写相关方法，最后通过 RecyclerView.addItemDecoration 设置。

- getItemOffsets(Rect outRect, View view, RecyclerView parent, State state)

  用于设置 ItemPosition 位置的 ItemView 上下左右额外间距

- onDraw(Canvas c, RecyclerView parent, State state)

  ItemView 绘制之前，调用此方法，此方法绘制的内容会在 ItemView 下方

- onDrawOver(Canvas c, RecyclerView parent, State state)

  ItemView 绘制完成后，调用此方法，此方法绘制的内容会在 ItemView 上方

自定义 ItemDecoration 实现分割线

```java
public class CustomItemDecoration extends RecyclerView.ItemDecoration {
    private static final float DIVIDER_HEIGHT = 1;
    private final float mDividerHeight;
    private final Paint mPaint;

    public CustomItemDecoration(Context context) {
        mPaint = new Paint();
        mPaint.setColor(Color.RED);
        mPaint.setAntiAlias(true);
    }

    @Override
    public void getItemOffsets(Rect outRect, View view, RecyclerView parent, RecyclerView.State state) {
        // 设置间距
        int layoutPosition = parent.getChildViewHolder(view).getAdapterPosition();
        if (layoutPosition != 0) {
            outRect.top = DIVIDER_HEIGHT;
            mDividerHeight = DIVIDER_HEIGHT;
        }
    }

    @Override
    public void onDraw(Canvas c, RecyclerView parent, RecyclerView.State state) {
        // 绘制分割线
        final int left = parent.getPaddingLeft();
        final int right = parent.getMeasuredWidth() - parent.getPaddingRight();
        final int count = parent.getChildCount();
        for (int i = 0; i < count; i++) {
            View child = parent.getChildAt(i);
            int position = parent.getChildAdapterPosition(child);
            if(position == 0){
                continue;
            }
            float dividerTop = child.getTop() - mDividerHeight;
            float dividerBottom = child.getBottom;
            float dividerLeft = parent.getPaddingLeft();
            float dividerRight = parent.getMeasuredWidth() - parent.getPaddingRight();
            c.drawRect(dividerLeft, dividerTop, dividerRight, dividerBottom, mPaint);
        }
    }

    @Override
    public void onDrawOver(Canvas c, RecyclerView parent, RecyclerView.State state) {
    }
}
```



[RecyclerView 源码分析(一) - RecyclerView的三大流程 - 简书 (jianshu.com)](https://www.jianshu.com/p/61fe3f3bb7ec)

[RecyclerView源码解析 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/263841552)

[抽丝剥茧RecyclerView - LayoutManager - 掘金 (juejin.cn)](https://juejin.cn/post/6844903924256735239)

[RecyclerView 各个 API 功能探索，超全 － 小专栏 (xiaozhuanlan.com)](https://xiaozhuanlan.com/topic/3102489765)

[RecyclerView 源码分析(三) - RecyclerView的缓存机制 - 简书 (jianshu.com)](https://www.jianshu.com/p/efe81969f69d)

[RecyclerView之ItemDecoration讲解及高级特性实践 (qq.com)](https://mp.weixin.qq.com/s/XUB1DrCG9-lZRXMMgIdrNw)

[RecyclerView 源码分析(四) - RecyclerView的动画机制 - 简书 (jianshu.com)](https://www.jianshu.com/p/65523b2ce15b)

[RecyclerView 源码分析(五) - Adapter的源码分析 - 简书 (jianshu.com)](https://www.jianshu.com/p/bdd9f4bdd90a)

[RecyclerView 扩展(一) - 手把手教你认识ItemDecoration - 简书 (jianshu.com)](https://www.jianshu.com/p/20851e4e32a7)

[RecyclerView扩展(六) - RecyclerView平滑滑动的实现原理 - 简书 (jianshu.com)](https://www.jianshu.com/p/c2e7d8a1ec5c)

