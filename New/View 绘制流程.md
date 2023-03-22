# View 绘制流程

## Paint 设置

在 Canvas 绘制时，需要一个 Paint 类型的参数。它控制了 Canvas 在绘制时的各种样式效果。Paint 的 API 可以分为：

- 设置颜色
- 设置效果
- 绘制文字相关设置
- 获取绘制的 Path

### 设置颜色

Paint 有三层颜色的处理：

- 直接设置颜色
- 设置 ColorFilter
- 设置 Xfermode

#### 直接设置颜色

Paint 直接设置颜色有以下方法：

- setColor(int color)

- setARGB(int a,int r, int g, int b)

- setShader(Shader shader)

  当设置了 Shader 后，前两个方法设置的颜色就会失效，Android 中 shader 有以下几个子类：

  - LinearGradient：线性渐变
  - RadiaGradient：辐射渐变
  - SweepGradient：扫描渐变
  - BitmapShader：图像着色器
  - ComposeShader：混合着色器

#### 设置 ColorFilter

Paint.setColorFilter

#### 设置 Xformade

Paint.setXfermode(Xfermode xfermode) 设置的是 Canvas  绘制的内容和绘制区域已存在图像的结合策略。Xfermode 只有一个子类 PorterDuffXfermode，已经绘制上的内容作为 destination，将要绘制的内容是 source：

| Mode 形式 |              实际效果               |
| :-------: | :---------------------------------: |
|    XXX    |            显示 XXX 图像            |
|  XXX_IN   | 显示两者相交的图像，采用 XXX 的颜色 |
|  XXX_OUT  |       显示 XXX 没有相交的部分       |
| XXX_OVER  |      XXX 显示在另一个图像之上       |
| XXX_ATOP  | 显示非XXX图像，相交部分采用XXX颜色  |
|   CLEAR   |              都不显示               |
|    XOR    |      显示两个图像不相交的部分       |

### 设置效果

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
- Paint.setMaskFilter：在绘制内容上方的附加效果
  - BlurMaskFilter：模糊效果
  - EmbossMaskFilter：浮雕效果

### 绘制文字相关设置

Paint 对文字绘制的辅助有两类方法：显示效果和测量文字尺寸

#### 文字效果

- Paint.setTextSize(float textSize)：设置文字大小
- Paint.setTypeface(Typeface typeface)：设置文字字体
- Paint.setFakeBoldText(boolean fakeBoldText)：是否使用伪粗体
- Paint.setStrikeThruText(boolean strikeThruText) 开启删除线
- Paint.setUnderlineText(boolean underlineText) 开启下划线
- Paint.setTextSkewX(float skewX) 设置文字横向错切角度
- Paint.setTextScaleX(float scaleX) 设置文字横向放缩
- Paint.setLetterSpacing(float letterSpacing) 设置字符间距
- Paint.setTextAlign(Paint.Align align) 设置文字的对齐方式

#### 文字测量

- float Paint.getFontSpacing()

  获取推荐的两行文字的行距，实际上就是 baseline 间的距离。这个值是系统根据文字的字体和字号自动计算的。它的作用是当要手动绘制多行文字（而不是使用 StaticLayout）的时候，可以在换行的时候给 y 坐标加上这个值来下移文字。

- FontMetrics Paint.getFontMetrics()

  根据 Paint 的当前字体和字号，获取 top、ascent、descent、bottom 这些数值的推荐值

- Paint.getTextBound(String text, int start, int end, Rect bounds)

  参数里，text 是要测量的文字，start 和 end 分别是文字的起始和结束位置，bounds 是存储文字显示范围的对象，方法在测算完成之后会把结果写进 bounds。

- float Paint.measureText(String text)

  测量文字的宽度并返回，对于同一串字符，此方法比 getTextBound 的返回值大一点。

- Paint.getTextWidths(String text, float[] widths)

  获取字符串中每个字符的宽度，并把结果填入参数 widths，相当于 measureText 的快捷方法，等价于对字符串中的每个字符分别调用 measureText。

- int Paint.breakText(String text, boolean measureForwards, float maxWidth, float[] measuredWidth)

  这个方法也是用来测量文字宽度的，但和 measureText() 的区别是，breakText() 是在给出宽度上限的前提下测量文字的宽度，如果文字的宽度超过了上限，就会在临近限制的位置截断文字。measureForwards 表示测量方向（true 是从左向右），maxWidth 是宽度上限，measuredWidth 用于接收测量截取的文字宽度数值。

### 获取绘制的 Path

#### getFillPath(Path src, Path dst)

Paint 在经过一系列设置后，实际上 drawPath 绘制的内容要算上线条的宽度和 PathEffect。方法的参数里，src 是原 Path ，而 dst 就是实际 Path 的保存位置。 getFillPath(src, dst) 会计算出实际 Path，然后把结果保存在 dst 里。

#### getTextPath(String text, int start, int end, float x, float y, Path path)

文字的绘制在实际绘制时，会被转化为图形，对图形进行绘制，方法就是获取目标文字所对应的 Path。

## Canvas 绘制

### 坐标系

Canvas 在二维环境和三维环境下有着不同的坐标系：

- 二维坐标系下，顺时针旋转为正，逆时针旋转为负。
- 三维坐标系下，通常是绕着某个轴进行旋转。我们从某个轴的正方向望去，绕这个轴顺时针旋转为正，逆时针旋转为负。

### 绘制基本图形

Canvas 绘制颜色和一些基本图形：

- drawCircle：绘制圆形图案
- drawOval：绘制椭图案
- drawRect：绘制矩形图案
- drawRoundRect：绘制圆角矩形图案
- drawPoint：绘制点
- drawLine：绘制直线
- drawArc：绘制弧线或者扇形
- drawColor：在整个绘制区域填充上指定颜色

### 绘制 Path

对于不规则的图案，需要先描述 Path 对象的路径，然后调用 Canvas.drawPath 来完成。Path 可以描述直线、二次曲线、三次曲线、圆等图形。

Path 添加路径的方法有三种：

- 直接添加图形
- 画线连成图形
- 辅助计算，如设置填充模式

#### 直接添加图形

Path 可以直接添加图形：

- addCircle(float x, float y, float radius, Direction dir)：添加圆形路径
- addOval(float left, float top, float right, float bottom, Direction dir) / addOval(RectF oval, Direction dir)：添加椭圆
- addRect(float left, float top, float right, float bottom, Direction dir)：添加矩形
- addRoundRect(float left, float top, float right, float bottom, float[] radii, Direction dir)：添加圆角矩形
- addPath(Path path)：添加另一个 Path

最后一个参数 Direct 是指路径的方向，有顺时针CW和逆时针CCW 两个选项。它在 Paint.Style 设置为 FILL 或者 FILL_AND_STROKE，且图形出现相交时判断填充范围。

#### 画线连成图形

Path 可以画线：

- lineTo/Path.rLineTo ：从当前位置添加一条直线路径，r前缀的方法使用的是相对位置
- quadTo/Path.rQuadTo：添加二次贝塞尔曲线
- cubicTo/Path.rCubicTo：添加三次贝塞尔曲线
- arcTo：添加弧形路径
- moveTo：移动到下次绘制的起点
- close：封闭当前的子图形

#### 辅助计算

Path.setFillType(Path.FillType) 设置两个相交 Path 如何进行绘制，其中 INVERSE 前缀的两个是前两个的反色：

- EVEN_ODD
- WINDING
- INVERSE_EVEN_ODD
- INVERSE_WINDING

EVEN_ODD 称为奇偶原则：对于平面中任意一点，向任意方向发射一条线，如果射线和图形相交次数是奇数，则这个点被认为在图形内部，需要被涂色；如果是偶数，则在外部，不需要涂色。

WINDING 称为非零环绕数原则，它与 Direction 类有关，Direction 有两种方向，一种是 CW(clock wise)顺时针方向、一种是 CCW(counter clock wise)逆时针方向：对于平面上任意一点，向任意方向发射一条线，射线与顺时针相交计数加1，与逆时针相交计数减1，如果结果不为0，被认为在图形内部，需要被涂色；如果等于0，则在外部，不需要涂色。

#### PathMeasure

pathMeasure 是一个用来测量 Path 的类，主要有以下的功能：

- setPath(Path path, boolean forceClosed)：关联一个Path
- isClosed()：是否闭合
- getLength()：获取Path的长度
- nextContour()：跳转到下一个轮廓
- getPosTan(float distance, float[] pos, float[] tan)：获取指定长度的位置坐标及该点切线值
- getMatrix(float distance, Matrix matrix, int flags)：获取指定长度的位置坐标及该点Matrix

### 绘制文字

Canvas 绘制文字通常有两个方法：

- Canvas.drawText(String text, float x, float y, Paint paint)：绘制文字
- Canvas.drawTextOnPath(String text, Path path, float hOffset, float vOffset, Paint paint)

由于 Canvas.drawText 只能绘制单行文字，不能换行，如果需要绘制多行文字可以借助 StaticLayout 来实现文字设置宽度上限来让文字自动换行，也会在 \n  处主动换行：

- StaticLayout.draw(Canvas canvas)

文字在排印方面有 5 条线来表示文字位置，他们从上到下分别是：top、ascent、baseline、descent、bottom。baseline 是文字显示的基准线

ascent/descent 的作用是限制普通字符的顶部和底部范围，具体到 Android 绘制中，ascent 的值是它与 baseline 的相对位移，数值为负因为它在 baseline 上方）；descent 的值是它与 baseline 的相对位移，数值为正（因为它在 baseline 下方）

top/bottom 的作用是所有字形的顶部和底部范围，具体到 Android 绘制中，top 的值是它与 baseline 的相对位移，数值为负（因为它在 baseline 上方）；bottom 的值是它与 baseline 的相对位移，数值为正（因为它在 baseline 下方）

![image-20230322214150020](C:\Users\24594\AppData\Roaming\Typora\typora-user-images\image-20230322214150020.png)

### 绘制 Bitmap

绘制 Bitmap 就是把这个 Bitmap 中的像素内容复制出来：

- drawBitmap(Bitmap bitmap, float left, float top, Paint paint)

#### Xformode

#### Bitmap 和 Drawable

Drawable 转 Bitmap

```java
public Bitmap drawableToBitmap(Drawable drawable) {
    if (drawable == null) {
        return null;
    }
    if (drawable instanceof BitmapDrawable) {
        return ((BitmapDrawable) drawable).getBitmap();
    }
    int width = drawable.getIntrinsicWidth();
    int height = drawable.getIntrinsicHeight();
    Bitmap bitmap = Bitmap.createBitmap(width, height, Bitmap.Config.ARGB_8888);
    Canvas canvas = new Canvas(bitmap);
    drawable.setBounds(0, 0, width, height);
    drawable.draw(canvas);
    return bitmap;
}
```

Bitmap 转 Drawable

```java
Drawable drawable = new BitmapDrawable(getResource(), bitmap);
```

### 范围裁切

Canvas 的范围裁切有两个方法：clipRect() 和 clipPath() 。裁切方法之后的绘制代码，都会被限制在裁切范围内。

在裁切的前后需要注意使用 Canvas.save() 和 Canvas.restore() 来保存和恢复绘制的内容。

### 几何变换

Canvas 的几何变换大概分为三类： Canvas（二维变换）、Matrix（二维变换） 和 Camera（三维变换）

#### 使用 Canvas 来做常见的二维变换

- Canvas.translate(float dx, float dy) 横向和纵向的位移。
- Canvas.rotate(float degrees, float px, float py) 旋转
- Canvas.scale(float sx, float sy, float px, float py) 横向和纵向的放缩倍数
- Canvas.skew(float sx, float sy) x 方向和 y 方向的错切系数

#### 使用 Matrix 来做常见和不常见的二维变换

使用 Matrix 做常见变换的流程是：

1. 创建 Matrix 对象
2. 调用 Matrix 的 pre/postTranslate pre/postRotate pre/postScale pre/postSkew 设置几何变换
3. 使用 Canvas.setMatrix(matrix) 或 Canvas.concat(matrix) 来把几何变换应用到 Canvas

setMatrix 是用 Matrix 直接替换 Canvas 当前的变换矩阵，concat 是基于当前 Canvas 的变换，再叠加上 Matrix 的变换。

#### 使用 Camera 来做三维变换

Camera 做三维变换主要有旋转和移动相机。

- Camera 旋转

  Camera.rotateX/Camera.rotateY/Camera.rotateZ 是相应的旋转方法，需要注意的是调用前后需要使用Canvas.save() 和 Canvas.restore() 来保存和恢复绘制的内容。

- 移动相机 Camera.setLocation(x, y, z)

  参数的单位不是像素，而是 inch英寸。Android 中 inch 和像素的换算单位写死是 72。在 Camera 中，相机的默认位置是 (0, 0, -8)（英寸）。8 x 72 = 576，所以它的默认位置是 (0, 0, -576)（像素）。如果绘制的内容过大，当它翻转起来的时候，就有可能出现图像投影过大的「糊脸」效果。而且由于换算单位被写死成了 72 像素，而不是和设备 dpi 相关的，所以在像素越大的手机上，这种「糊脸」效果会越明显。

  而使用 setLocation 方法来把相机往后移动，就可以修复这种问题。

  ```java
  DisplayMetrics displayMetrics = getResources().getDisplayMetrics();
  float newZ = -displayMetrics.density * 6;
  mCamera.setLocation(0, 0, newZ);
  ```

### 保存和恢复

#### 状态的保存和恢复

- save()
- restore()
- restoreToCount()

save 和 restore 一般成对的出现，save 可以保存 Canvas 当前的状态，随后进行平移，裁剪等一系列改变 Canvas 的操作，最后使用 restore 将 Canvas 还原成 save 时候的状态。

save() 返回一个 int 值，该值是这次保存的标志，这个值可以作为 restoreToCount() 的参数，从而指定 Canvas 返回到某个指定的 save 状态。

#### 图层的保存和恢复

saveLayer 可以为 Canvas 创建一个新的透明图层，在新的图层上绘制，不会直接绘制到屏幕上，而会在 restore 之后，绘制到上一个图层或者屏幕上。

saveLayer 方法参数 saveFlags 代表了需要保存哪方面的内容：

- MATRIX_SAVE_FLAG：只保存图层的 matrix 矩阵
- CLIP_SAVE_FLAG：只保存图层裁剪大小信息
- HAS_ALPHA_LAYER_SAVE_FLAG：表明该图层有透明度，和下面的标识冲突，都设置时以下面的标志为准
- FULL_COLOR_LAYER_SAVE_FLAG：完全保留该图层颜色（和上一图层合并时，清空上一图层的重叠区域，保留该图层的颜色）
- CLIP_TO_LAYER_SAVE_FLAG：创建图层时，会把canvas（所有图层）裁剪到参数指定的范围，如果省略这个flag将导致图层开销巨大（实际上图层没有裁剪，与原图层一样大）
- ALL_SAVE_FLAG：保存所有信息
