自定义 View 的绘制主要是在 onDraw() 方法中完成的，另外还涉及到 drawBackground() dispatchDraw() onDrawForeground() 方法，它们统一在 draw() 方法中调度：

```java
public void draw(Canvas canvas) {
    final int privateFlags = mPrivateFlags;
    final boolean dirtyOpaque = (privateFlags & PFLAG_DIRTY_MASK) == PFLAG_DIRTY_OPAQUE &&
            (mAttachInfo == null || !mAttachInfo.mIgnoreDirtyState);
    mPrivateFlags = (privateFlags & ~PFLAG_DIRTY_MASK) | PFLAG_DRAWN;
  
    // Step 1, draw the background, if needed
    int saveCount;
    if (!dirtyOpaque) {
        drawBackground(canvas);
    }
    // skip step 2 & 5 if possible (common case)
    final int viewFlags = mViewFlags;
    boolean horizontalEdges = (viewFlags & FADING_EDGE_HORIZONTAL) != 0;
    boolean verticalEdges = (viewFlags & FADING_EDGE_VERTICAL) != 0;
    if (!verticalEdges && !horizontalEdges) {
        // Step 3, draw the content
        if (!dirtyOpaque) onDraw(canvas);
        // Step 4, draw the children
        dispatchDraw(canvas);
        drawAutofilledHighlight(canvas);
        // Overlay is part of the content and draws beneath Foreground
        if (mOverlay != null && !mOverlay.isEmpty()) {
            mOverlay.getOverlayView().dispatchDraw(canvas);
        }
        // Step 6, draw decorations (foreground, scrollbars)
        onDrawForeground(canvas);
        // Step 7, draw the default focus highlight
        drawDefaultFocusHighlight(canvas);
        if (debugDraw()) {
            debugDrawFocus(canvas);
        }
        // we're done...
        return;
    }
```

可以看到，整个 draw 的调度过程中最先开始的是 drawBackground() 它负责绘制 View 的背景，接着就会调用 onDraw() 方法来完成 View 本体的绘制，然后通过调用 dispatchDraw() 方法通知子 View 开始绘制，最后会调用 onDrawForeground() 绘制前景。

有一些需要的地方，ViewGroup 没有背景时会绕过 draw 而直接调用 dispatchDraw 方法，所以 ViewGroup 绘制往往是写在 dispatchDraw() 中。

通常绘制 View 都发生在 onDraw 中，从使用角度可以分为几大类

##### Canvas 基本使用

###### Canvas.drawXxx()

Canvas 绘制颜色和一些基本图形，Android 已经提供了相应的 api。

- drawCircle：绘制圆形图案
- drawOval：绘制椭图案
- drawRect：绘制矩形图案
- drawRoundRect：绘制圆角矩形图案
- drawPoint：绘制点
- drawLine：绘制直线
- drawArc：绘制弧线或者扇形

###### Canvas.drawPath()

对于不规则的图案，需要借助 Path 类来完成。首先我们需要先描述 Path 对象的路径，最后调用 Canvas.drawPath 方法。Path 添加路径的分为：

- Path.addXxx 直接添加子图形

  - Path.addCircle(float x, float y, float radius, Direction dir) 添加圆形路径
  - Path.addOval：添加椭圆路径
  - Path.addRoundRect：添加圆角矩形路径
  - Path.addPath：添加一个新的 Path 路径

- Path.xxxTo 直接画线

  - Path.lineTo/Path.rLineTo ：从当前位置添加一条直线路径，r前缀的方法使用的是相对位置，不带的使用的绝对位置
  - Path.quadTo/Path.rQuadTo：添加二次贝塞尔曲线
  - Path.cubicTo/Path.rCubicTo：添加三次贝塞尔曲线
  - Path.arcTo：添加弧形路径
  - Path.moveTo：移动下次绘制的起点

- Path.setFillType(Path.FillType) 设置填充方式

  FillType 会设置两个相交 Path 如何进行绘制，取值有四个。其中 INVERSE 前缀的两个是前两个的反色：

  - EVEN_ODD
  - WINDING
  - INVERSE_EVEN_ODD
  - INVERSE_WINDING

  EVEN_ODD 称为奇偶原则：对于平面中任意一点，向任意方向发射一条线，如果射线和图形相交次数是奇数，则这个点被认为在图形内部，需要被涂色；如果是偶数，则在外部，不需要涂色。

  WINDING 称为非零环绕数原则，它与前面所列出的 Direction 类有关，Direction 有两种方向，一种是 CW(clock wise)顺时针方向、一种是 CCW(counter clock wise)逆时针方向：对于平面上任意一点，向任意方向发射一条线，射线与顺时针相交计数加1，与逆时针相交计数减1，如果结果不为0，被认为在图形内部，需要被涂色；如果等于0，则在外部，不需要涂色。

###### Canvas 绘制颜色

除了 Canvas.drawColor/Canvas.drawBitmap 可以直接通过 Canvas 控制内容的颜色，大多数的效果都需要 Paint 的参与，详见 Paint 基本使用-设置颜色。

###### Canvas 绘制文字

##### Paint 基本使用

在 Canvas 绘制时，总需要一个 Paint 类型的入参。它实际上控制了 Canvas 在绘制时的各种样式效果。Paint 的功能可以分为

###### 设置颜色

前文提到 Canvas 绘制时除了 drawColor/drawBitmap，大部分颜色的设置都是通过 Paint 来设置的，Paint 设置颜色有以下方法

- Paint.setColor：直接设置颜色

- Paint.setARGB：直接设置颜色

- Paint.setShader：设置着色器

  Android 中 经常使用的 Shader 有：LInearGradient(线性渐变)、RadialGradient(辐射渐变)、sweepGradient(扫描渐变)、BitmapShader(图像着色器)、ComposeShader(混合着色器)

- Paint.setColorFilter

- Paint.setXfermode

  设置的是 Canvas  绘制的内容和绘制区域已存在图像的结合策略。

  PorterDuff.Mode 是用于指定两个图像共同绘制时的颜色策略的，它是一个 enum，不同的 Mode 指定不同的策略，一般来说都是后者作为 src 前者作为 dst 来进行结合。共有 17 种，分为 Alpha 合成和混合， Alpha合成共有 12 种，规律如下：

  | Mode 形式 |              实际效果               |
  | :-------: | :---------------------------------: |
  |    XXX    |            显示 XXX 图像            |
  |  XXX_IN   | 显示两者相交的图像，采用 XXX 的颜色 |
  |  XXX_OUT  |       显示 XXX 没有相交的部分       |
  | XXX_OVER  |      XXX 显示在另一个图像之上       |
  | XXX_ATOP  | 显示非XXX图像，相交部分采用XXX颜色  |
  |   CLEAR   |              都不显示               |
  |    XOR    |      显示两个图像不相交的部分       |

###### 设置效果

- Paint.setAntiAlias：设置是否抗锯齿
- Paint.setStyle：设置线条风格还是填充风格
- Paint,setStrokeWidth：设置线条宽度，单位是像素
- Paint.setStrokeCap：设置线头的形状，有 BUTT 平头，SQUARE 方头、ROUND 圆头
- Paint.setStrokeJoin：设置拐点形状，有 MITER 尖头、BEVEL平角、ROUND 圆角
- Paint.setStrokeMiter：设置 MITER  型拐角的延长线的最大值
- Paint.setDither：设置是否开启图像抖动
- Paint.setFilterBitmap：设置是否使用双线性过滤来绘制 Bitmap
- Paint.setPathEffect：设置图形的轮廓显示效果
  - CornerPathEffect：把所有拐角变为圆角
  - DiscretePathEffect：把线条进行随机的偏离
  - DashPathEffect：使用虚线绘制线条
  - PathDashPathEffect：使用 Path 来绘制线条路径
  - SumPathEffect：组合效果，分别根据两个 PathEffect 绘制两条效果
  - ComposePathEffect：组合效果，对线条按顺序使用效果，只绘制一条效果
- Paint.setShadowLayer：在绘制内容下面加一层阴影

##### Canvas 的范围裁切和几何变换