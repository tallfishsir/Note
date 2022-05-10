# Android 位移动画

## View 参数

### 顶点参数

View 的位置是由它的四个顶点来决定的，分别对应 View 的四个属性：left、top、right、bottom。需要注意的是这些坐标都是相对于 View 的父容器来说的，

这四个属性在 View 的源码中对应 mLeft、mTop、mRight、mBottom 四个成员变量：

```java
public final int getLeft() {
    return mLeft;
}

public final int getTop() {
    return mTop;
}

public final int getRight() {
    return mRight;
}

public final int getBottom() {
    return mBottom;
}
```

由此，可以得出 View 的宽高和坐标的关系：

```java
public final int getWidth() {
    return mRight - mLeft;
}

public final int getHeight() {
    return mBottom - mTop;
}
```

### 位置参数

Android 3.0 开始，View 增加了 x、y、translationX、translationY 参数，这几个参数都是相对于 View 的父容器来说的。

x、y 是 View 左上角的坐标

translationX、translationY View 左上角相对于父容器的偏移量，View 提供了 get/set 方法，初始默认值是 0。

这几个参数和顶点参数的换算关系是：

- x = left + translationX
- y = top + translationY

```java
public float getX() {
    return mLeft + getTranslationX();
}

public float getY() {
    return mTop + getTranslationY();
}
```

### 滚动参数

View 内部还有两个属性 scrollX、scrollY，用于改变 View 的内容的位置，但不能改变 View 在布局中位置

> scrollX：View 左边缘和 View 内容左边缘在水平方向的距离
>
> scrollY：View 上边缘和 View 内容上边缘在水平方向的距离

View 边缘是指 View 的位置，由 View 的四个顶点位置组成

View 内容边缘是指 View 中的内容的边缘。

## MotionEvent 参数

在手指接触屏幕后，会产生一系列的事件，产生对应的 MotionEvent 对象

### 相对坐标

MotionEvent 的 getX() 和 getY() 返回的是相对于当前 View 左上角 的 （x，y）坐标。

### 绝对坐标

MotionEvent 的 getRawX() 和 getRawY() 返回的是相对于屏幕左上角 的 （x，y）坐标。

## 常见位移动画实现方式

### scrollTo/scrollBy

```
public void scrollBy(int x, int y) {
    scrollTo(mScrollX + x, mScrollY + y);
}

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
```

scrollTo 是基于所传递参数的绝对滑动。

scrollBy 是基于当前位置的相对滑动。

### 视图动画

```java
TranslateAnimation animation = new TranslateAnimation(...)
view.startAnimation(animation)
```

### 属性动画

```java
view.animate().translationX(...).start()
```

### 改变布局参数

```java
val params = view.layoutParams as ViewGroup.MarginLayoutParams
params.updateMargins(...)
requestLayout()
```

