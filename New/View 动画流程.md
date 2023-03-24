# View 动画流程

Android 动画分为两大类：视图动画和属性动画，视图动画又分为逐帧动画和补间动画

## 视图动画

视图动画的作用对象都是 View，它存在三个问题：

- 作用对象有限，只能作用在视图 View 上
- 仅改变了视觉效果，无法改变 View 的属性，导致点击事件等无法跟随变化位置
- 动画效果单一

### 逐帧动画

逐帧动画是把动画拆分为帧的形式，按照顺序播放一组预先定义好的图片。

#### 方式一

在 drawable 包下创建动画效果的 xml 文件，然后在代码中加载并开启动画。

```java
res/drawable/frame_animation.xml
<?xml version="1.0" encoding="utf-8"?>
<animation-list xmlns:android="http://schemas.android.com/apk/res/android"
    android:oneshot="true"> // 设置是否只播放一次，默认为false
    <item android:drawable="@drawable/a0" android:duration="100"/>
    <item android:drawable="@drawable/a1" android:duration="100"/>
    <item android:drawable="@drawable/a2" android:duration="100"/>
    <item android:drawable="@drawable/a3" android:duration="100"/>
    <item android:drawable="@drawable/a4" android:duration="100"/>
    <item android:drawable="@drawable/a5" android:duration="100"/>
    <item android:drawable="@drawable/a6" android:duration="100"/>
    <item android:drawable="@drawable/a7" android:duration="100"/>
    <item android:drawable="@drawable/a8" android:duration="100"/>
    <item android:drawable="@drawable/a9" android:duration="100"/>
</animation-list>

view.setImageResource(R.drawable.frame_animation);
animationDrawable = (AnimationDrawable) iv.getDrawable();
animationDrawable.start();
```

#### 方式二

直接从 drawable 文件夹中获取动画资源，生成 AnimationDrawable 对象，然后开启动画。

```
AnimationDrawable animationDrawable = new AnimationDrawable();
for (int i = 0; i <= 25; i++) {
    int id = getResources().getIdentifier("a" + i, "drawable", getPackageName());
    Drawable drawable = getResources().getDrawable(id);
    animationDrawable.addFrame(drawable, 100);
}

view.setImageDrawable(animationDrawable);
animationDrawable = (AnimationDrawable) iv.getDrawable();
animationDrawable.start();
```

### 补间动画

补间动画通过确定开始的视图样式和结束的视图样式，中间动画变化过程由系统补全来完成，包含了平移动画、缩放动画、旋转动画、透明度动画。

#### 方式一

在 anim 包下新建动画效果 xml 文件，然后代码中加载并开启动画

```java
res/anim/view_animation.xml
<?xml version="1.0" encoding="utf-8"?>
<translate xmlns:android="http://schemas.android.com/apk/res/android"
<scale xmlns:android="http://schemas.android.com/apk/res/android"
<rotate xmlns:android="http://schemas.android.com/apk/res/android"
<alpha xmlns:android="http://schemas.android.com/apk/res/android"
    android:duration="3000" // 动画持续时间（ms），必须设置，动画才有效果
    android:startOffset ="1000" // 动画延迟开始时间（ms）
    android:fillBefore = “true” // 动画播放完后，视图是否会停留在动画开始的状态，默认为true
    android:fillAfter = “false” // 动画播放完后，视图是否会停留在动画结束的状态，优先于fillBefore值，默认为false
    android:fillEnabled= “true” // 是否应用fillBefore值，对fillAfter值无影响，默认为true
    android:repeatMode= “restart” // 选择重复播放动画模式，restart代表正序重放，reverse代表倒序回放，默认为restart|
    android:repeatCount = “0” // 重放次数（所以动画的播放次数=重放次数+1），为infinite时无限重复
    android:interpolator = @[package:]anim/interpolator_resource // 插值器
           
    android:fromXDelta="0" // translate特有，水平方向x 移动的起始值
    android:toXDelta="500" // translate特有，水平方向x 移动的结束值
    android:fromYDelta="0" // translate特有，竖直方向y 移动的起始值
    android:toYDelta="500" // translate特有，竖直方向y 移动的结束值
        
    android:fromXScale="0.0" // scale特有，水平方向X的起始缩放倍数
    android:toXScale="2"  // scale特有，水平方向X的结束缩放倍数
	android:fromYScale="0.0" // scale特有，竖直方向Y的起始缩放倍数
    android:toYScale="2" // scale特有，竖直方向Y的结束缩放倍数
	android:pivotX="50%" // scale特有，缩放轴点的x坐标
    android:pivotY="50%" // scale特有，缩放轴点的y坐标
	
	android:fromDegrees="0" // rotate特有，动画开始的旋转角度(正数 = 顺时针，负数 = 逆时针)
    android:toDegrees="270" // rotate特有，动画结束的旋转角度(正数 = 顺时针，负数 = 逆时针)
    android:pivotX="50%" // rotate特有，缩放轴点的x坐标
    android:pivotY="0" // rotate特有，缩放轴点的y坐标
	
	android:fromAlpha="1.0" // alpha特有，动画开始时视图的透明度(取值范围: -1 ~ 1)
    android:toAlpha="0.0"// alpha特有，动画结束时视图的透明度(取值范围: -1 ~ 1)
/> 

代码中执行：
Animation animation = AnimationUtils.loadAnimation(this, R.anim.view_animation);
view.startAnimation(animation);
```

#### 方式二

创建对应的 Animation 子类，设置不同的效果，开启动画

```java
Animation animation = new TranslateAnimation(0，500，0，500);
Animation animation = new ScaleAnimation(0,2,0,2,Animation.RELATIVE_TO_SELF,0.5f,Animation.RELATIVE_TO_SELF,0.5f);
// pivotXType = Animation.ABSOLUTE:缩放轴点的x坐标 =  View左上角的原点 在x方向 加上 pivotXValue数值的点(y方向同理)
 // pivotXType = Animation.RELATIVE_TO_SELF:缩放轴点的x坐标 = View左上角的原点 在x方向 加上 自身宽度乘上pivotXValue数值的值(y方向同理)
 // pivotXType = Animation.RELATIVE_TO_PARENT:缩放轴点的x坐标 = View左上角的原点 在x方向 加上 父控件宽度乘上pivotXValue数值的值 (y方向同理)
Animation animation = new RotateAnimation(0,270,Animation.RELATIVE_TO_SELF,0.5f,Animation.RELATIVE_TO_SELF,0.5f);

animation.setDuration(3000);
view.startAnimation(animation);
```

## 属性动画

View 的属性动画通常有两种方法：ViewPropertyAnimator 和 ObjectAnimator。这两种方式底层都是通过 ValueAnimator 改变 View 的属性然后通知系统重绘来实现效果的。

### ValueAnimator

ValueAnimator 是属性动画中最核心的类，本质是一种对值的变化和监听机制，并没有对 View 的某个属性进行操作。整体流程是：

- 通过 Choregrapher.postFrameCallback 监听 VSync 信号
- 实现 AnimationFrameCallback.doAnimationFrame 接口
- 当 VSync 信号到达时，回调 doAnimationFrame() 并在 animateValue() 中回调 onAnimationUpdate
- 当 VSync 信号到达时，再次 postFrameCallback 监听下一次信号

```java
//ValueAnimator.java
public void start() {
    start(false);
}

private void start(boolean playBackwards) {
    if (Looper.myLooper() == null) {
        throw new AndroidRuntimeException("Animators may only be run on Looper threads");
    }
    ...
    //开启动画
    addAnimationCallback(0);
    if (mStartDelay == 0 || mSeekFraction >= 0 || mReversing) {
        //执行动画的一些回调
        startAnimation();
        if (mSeekFraction == -1) {
            setCurrentPlayTime(0);
        } else {
            setCurrentFraction(mSeekFraction);
        }
    }
}

private void addAnimationCallback(long delay) {
    //Choregrapher.postFrameCallback 监听 VSync 信号
    //第一个参数this，是指实现的AnimationFrameCallback.doAnimationFrame
    getAnimationHandler().addAnimationFrameCallback(this, delay);
}

public final boolean doAnimationFrame(long frameTime) {
	...
	mLastFrameTime = frameTime;
    final long currentTime = Math.max(frameTime, mStartTime);
    //计算fraction和values
    boolean finished = animateBasedOnTime(currentTime);
    if (finished) {
        //执行动画的一些回调
        endAnimation();
    }
    return finished;
}

boolean animateBasedOnTime(long currentTime) {
    boolean done = false;
    if (mRunning) {
        ...
        animateValue(currentIterationFraction);
    }
    return done;
}

void animateValue(float fraction) {
    //根据mInterpolator计算fraction
    fraction = mInterpolator.getInterpolation(fraction);
    mCurrentFraction = fraction;
    int numValues = mValues.length;
    for (int i = 0; i < numValues; ++i) {
        mValues[i].calculateValue(fraction);
    }
    //onAnimationUpdate回调
    if (mUpdateListeners != null) {
        int numListeners = mUpdateListeners.size();
        for (int i = 0; i < numListeners; ++i) {
            mUpdateListeners.get(i).onAnimationUpdate(this);
        }
    }
}

//AnimationHandler.java
public void addAnimationFrameCallback(final AnimationFrameCallback callback, long delay) {
    //Choregrapher.postFrameCallback监听VSync信号
    if (mAnimationCallbacks.size() == 0) {
        getProvider().postFrameCallback(mFrameCallback);
    }
    //保存ValueAnimator传来的AnimationFrameCallback
    if (!mAnimationCallbacks.contains(callback)) {
        mAnimationCallbacks.add(callback);
    }
}

private final Choreographer.FrameCallback mFrameCallback = new Choreographer.FrameCallback() {
    @Override
    public void doFrame(long frameTimeNanos) {
        //回调ValueAnimator的AnimationFrameCallback接口
        doAnimationFrame(getProvider().getFrameTime());
        //Choregrapher.postFrameCallback监听下一次VSync信号
        if (mAnimationCallbacks.size() > 0) {
            getProvider().postFrameCallback(this);
        }
    }
};

private void doAnimationFrame(long frameTime) {
    long currentTime = SystemClock.uptimeMillis();
    final int size = mAnimationCallbacks.size();
    for (int i = 0; i < size; i++) {
        final AnimationFrameCallback callback = mAnimationCallbacks.get(i);
        if (callback == null) {
            continue;
        }
        if (isCallbackDue(callback, currentTime)) {
            callback.doAnimationFrame(frameTime);
        }
    }
    cleanUpList();
}
```

#### Interpolator

在 ValueAnimator.animateValue() 中会根据 mInterpolator 来重新计算 fraction。Interpolator 可以通过 setInterpolator(Interpolator interpolator) 设置，常见的 Interpolator 有：

- AccelerateDecelerateInterpolator：先加速再减速
- LinearInterpolator：匀速
- AccelerateInterpolator：持续加速
- DecelerateInterpolator：持续减速
- AnticipateInterpolator：开始时回拉再动画
- OvershootInterpolator：结束时超过目标再恢复
- BounceInterpolator：结束时弹跳
- PathInterpolator：生成从(0,0)到(1,1)的 Path 完成

#### Listener

AnimatorListener 用于监听动画的状态，通知指示动画相关事件

- onAnimationStart：动画开始是执行
- onAnimationEnd：动画结束时执行
- onAnimationCancel：动画被取消时执行
- onAnimationRepeat：动画被 repeat 时执行

AnimatorUpdateListener 用于动画被更新时通知相关事件：

- onAnimationUpdate

### ViewPropertyAnimator

View 通过调用 animate() 获取 ViewPropertyAnimator 对象，然后调用不同的 API 完成不同的效果，动画会自动执行：

- translationX()/translationY()/translationZ()： x/y/z 轴的绝对坐标偏移
- translationXBy()/translationYBy()/translationZBy()：x/y/z 轴的相对位置偏移
- rotation()/rotationX()/rotationY()：设置绕着 z/x/y 轴旋转绝对角度
- rotationBy()/rotationXBy()/rotationYBy()：设置绕着 z/x/y 轴旋转再旋转相对角度
- scaleX()/scaleY()：设置横向/纵向绝对大小的放缩
- scaleXBy()/scaleYBy()：设置横向/纵向相对大小的放缩
- alpha()：设置透明度绝对大小
- alphaBy()：设置透明度相对大小

上面这些方法实现动画的流程是：

- 计算出 fromValue 和 deltaValue，然后将属性封装为 NameValueHolders 对象
- 创建 ValueAnimator 对象，设置 AnimatorUpdateListener 监听，然后开启动画
- 在 AnimatorUpdateListener 回调中更新 NameValueHolders 对应的属性

```java
public @NonNull ViewPropertyAnimator translationX(float value) {
    animateProperty(TRANSLATION_X, value);
    return this;
}

private void animateProperty(int constantName, float toValue) {
    float fromValue = getValue(constantName);
    float deltaValue = toValue - fromValue;
    animatePropertyBy(constantName, fromValue, deltaValue);
}

private void animatePropertyBy(int constantName, float startValue, float byValue) {
    //将改变的属性封装为NameValuesHolder
    NameValuesHolder nameValuePair = new NameValuesHolder(constantName, startValue, byVal
    mPendingAnimations.add(nameValuePair);
    mView.removeCallbacks(mAnimationStarter);
    //View.postOnAnimation()会调用Choreographer.postCallback下一个VSync信号执行
    mView.postOnAnimation(mAnimationStarter);
}

private Runnable mAnimationStarter = new Runnable() {
    @Override
    public void run() {
        startAnimation();
    }
};

private void startAnimation() {
    //创建一个ValueAnimator
    ValueAnimator animator = ValueAnimator.ofFloat(1.0f);
    ...
    //将NameValuesHolder list整合到mAnimatorMap中
    ArrayList<NameValuesHolder> nameValueList = (ArrayList<NameValuesHolder>) mPendingAnimations.clone();
    mAnimatorMap.put(animator, new PropertyBundle(propertyMask, nameValueList));
    ...
    //设置UpdateListener，每次更新mAnimatorMa中的属性数值
    animator.addUpdateListener(mAnimatorEventListener);
    animator.addListener(mAnimatorEventListener);
    //ValueAnimator.start()会调用Choreographer.postFrameCallback下一个VSync信号启动动画
    animator.start();
}
```

### ObjectAnimator

ObjectAnimator  是 ValueAnimator 的子类，继承了 start() 方法，常见创建对象的方法有：

- ofArgb(Object target, String propertyName, int... values)
- ofFloat(Object target, String propertyName, float... values)
- ofInt(Object target, String propertyName, int... values)

参数 target 和 propertyName 会反射到对象属性的 setter/getter 方法。参数 value 都会被封装为 PropertyValuesHolder 对象，最后全部保存在 mValuesMap 中。

ObjectAnimator 重写了 animateValue() 在其中增加了反射调用 mValuesMap 的 setter 来设置 target 的属性：

- 通过 Choregrapher.postFrameCallback 监听 VSync 信号
- 实现 AnimationFrameCallback.doAnimationFrame 接口
- 当 VSync 信号到达时，回调 doAnimationFrame() 并在 animateValue() 中设置属性
- 当 VSync 信号到达时，再次 postFrameCallback 监听下一次信号

```java
void animateValue(float fraction) {
    final Object target = getTarget();
    if (mTarget != null && target == null) {
        cancel();
        return;
    }
    super.animateValue(fraction);
    int numValues = mValues.length;
    for (int i = 0; i < numValues; ++i) {
        mValues[i].setAnimatedValue(target);
    }
}
```

需要注意的是，如果参数 propertyName 是自定义属性，需要将 setter/getter 设置可见，且 setter 里需要调用 invalidate() 来触发重绘。

#### PropertyValuesHolders/Keyframe

以上提到的几个方法都是单一的修改一个属性进行动画，如果需要同时对一个 View 的多个属性进行动画，可以使用 PropertyValuesHolder

```java
PropertyValuesHolder holder1 = PropertyValuesHolder.ofFloat("scaleX", 1);
PropertyValuesHolder holder2 = PropertyValuesHolder.ofFloat("scaleY", 1);
PropertyValuesHolder holder3 = PropertyValuesHolder.ofFloat("alpha", 1);

ObjectAnimator animator = ObjectAnimator.ofPropertyValuesHolder(view, holder1, holder2, holder3)
animator.start();
```

如果需要对一个属性动画，进行更细的定制，将一个动画属性拆分为多个阶段，可以使用 Keyframe：

```java
/ 在 0% 处开始
Keyframe keyframe1 = Keyframe.ofFloat(0, 0);
// 时间经过 50% 的时候，动画完成度 100%
Keyframe keyframe2 = Keyframe.ofFloat(0.5f, 100);
// 时间见过 100% 的时候，动画完成度倒退到 80%，即反弹 20%
Keyframe keyframe3 = Keyframe.ofFloat(1, 80);

PropertyValuesHolder holder = PropertyValuesHolder.ofKeyframe("progress", keyframe1, keyframe2, keyframe3);
ObjectAnimator animator = ObjectAnimator.ofPropertyValuesHolder(view, holder);
animator.start();
```

#### TypeEvalutor

当修改 View 的属性并不是基本数据类型，就需要调用 ofObject(Object target, String propertyName, TypeEvaluator evaluator, Object... values) 来进行动画。TypeEvaluator 是设置动画完成度到属性具体数值的计算公式。

```java
private class PointFEvaluator implements TypeEvaluator<PointF> {
   PointF newPoint = new PointF();

   @Override
   public PointF evaluate(float fraction, PointF startValue, PointF endValue) {
       float x = startValue.x + (fraction * (endValue.x - startValue.x));
       float y = startValue.y + (fraction * (endValue.y - startValue.y));

       newPoint.set(x, y);

       return newPoint;
   }
}

ObjectAnimator animator = ObjectAnimator.ofObject(view, "position",
        new PointFEvaluator(), new PointF(0, 0), new PointF(1, 1));
animator.start();
```

### AnimatorSet

AnimatorSet 可以让多个 Animator 互相配合运行，实现复杂的组合动画。

#### playTogether/playSequentially

AnimatorSet 可以使用以上两个方法，一次性添加所有的动画：

- playTogether：所有动画一起执行
- playTogether：所有动画按照添加顺序执行

#### Animator.Builder

AnimatorSet.play() 创建了 Builder 对象，Builder 可以控制动画的执行顺序和互相之间的依赖

-  with(Animator anim) 和前面动画一起执行
- before(Animator anim) 执行前面的动画后再执行anim动画
- after(Animator anim) 先执行anim动画再执行前面动画
- after(long delay) 延迟n毫秒之后执行动画



[Carson带你学Android：关于逐帧动画的使用都在这里了！_使用逐帧动画的原因_Carson带你学Android的博客-CSDN博客](https://blog.csdn.net/carson_ho/article/details/73087488)

[Carson带你学Android：手把手带你全面学习补间动画的使用！_安卓 补间动画使用_Carson带你学Android的博客-CSDN博客](https://carsonho.blog.csdn.net/article/details/120504213)

[Android 动画：这是一份详细 & 清晰的 动画学习指南 - 掘金 (juejin.cn)](https://juejin.cn/post/6844903601190486030)

[Android动画之AnimatorSet联合动画用法 - 简书 (jianshu.com)](https://www.jianshu.com/p/b2204968f3ff)