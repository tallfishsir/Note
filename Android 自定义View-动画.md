Android View 的动画可以分为 View Animation 和 Property Animation 两类。

#####  View Animation

View 的属性动画用到的了 ViewPropertyAnimator 类，它的使用方式：

1. 通过 View.animate() 创建 ViewPropertyAnimator 对象

2. 通过以下方法设置 View 动画

   |             ViewPropertyAnimator 方法              |                功能                 |
   | :------------------------------------------------: | :---------------------------------: |
   |    translationX()/translationY()/translationZ()    |     设置 x/y/z 轴的绝对坐标偏移     |
   | translationXBy()/translationYBy()/translationZBy() |     设置 x/y/z 轴的相对位置偏移     |
   |         rotation()/rotationX()/rotationY()         |    设置绕着 z/x/y 轴旋转绝对角度    |
   |      rotationBy()/rotationXBy()/rotationYBy()      | 设置绕着 z/x/y 轴旋转再旋转相对角度 |
   |                 scaleX()/scaleY()                  |     设置横向/纵向绝对大小的放缩     |
   |               scaleXBy()/scaleYBy()                |     设置横向/纵向相对大小的放缩     |
   |                      alpha()                       |         设置透明度绝对大小          |
   |                     alphaBy()                      |         设置透明度相对大小          |

3. 调用 ViewPropertyAnimator 的 start()/startDelay() 方法执行动画。

##### Property Animation

View 的属性动画用到的了 ObjectAnimator 类，它的使用方式：

1. 如果是自定义控件， 需要添加 setter/gettter 方法；
2. 用 ObjectAnimator.ofXxx() 创建 ObjectAnimator 对象；
3. 调用 ObjectAnimator 的 start()/startDelay() 方法执行动画。

##### 通用设置方法

###### setDuration(int duration) 设置动画时长

单位是毫秒

###### setInterpolator(Interpolator interpolator) 设置速度差值器 Interpolator

一些常用的 Interpolator 有：

- AccelerateDecelerateInterpolator 先加速再减速
- LinearInterpolator 匀速
- AccelerateInterpolator 持续加速
- DecelerateInterpolator 持续减速

###### ViewPropertyAnimator 设置监听器

- ViewPropertyAnimator.setListenter(AnimatorListener animatorListener)

  AnimatorListener 有 3 个回调方法：

  - onAnimationStart(Animator animation) 动画开始执行时被调用
  - onAnimationEnd(Animator animation) 动画结束时被调用
  - onAnimationCancel(Animator animation)  动画被通过 cancel() 方法取消时被调用

- ViewPropetyAnimator.setAnimationUpdate(ValueAnimator animation)

  当动画的属性更新时（不严谨的说，即每过 10 毫秒，动画的完成度更新时），这个方法被调用。

  方法的参数是一个 ValueAnimator，是 ViewPropertyAnimator 的内部实现。

###### ObjectAnimator 设置监听器

- ObjectAnimator.addListener(AnimatorListener animatorListener)

  AnimatorListener 有 4 个回调方法：

  - onAnimationStart(Animator animation) 动画开始执行时被调用
  - onAnimationEnd(Animator animation) 动画结束时被调用
  - onAnimationCancel(Animator animation)  动画通过 cancel() 方法取消时被调用
  - onAnimationRepeat(Animator animation) 动画通过 setRepeatMode() / setRepeatCount() 或 repeat() 方法重复执行时被调用

- ObjectAnimator.addUpdateListener(ValueAnimator animation)

  方法的参数是一个 ValueAnimator，ValueAnimator是 ObjectAnimator 的父类，也是 ViewPropertyAnimator 的内部实现。

- ObjectAnimator.addPauseListener()

##### Property Animation 针对复杂属性的动画

###### TypeEvaluator 估值器

ObjectAnimator 是 ValueAnimator 的子类，对于 ObjectAnimator.ofXxx() 生成的 ObjectAnimator 对象，他的变化规律是已经被设置的，如果需要不同的变化规律，可以通过 ValueAnimator .setEvaluator() 方法设置。

对于自定义估值器 TypeEvaluator 需要实现 evaluate 方法

```java
public class FloatEvaluator implements TypeEvaluator<Number> {
    public Float evaluate(float fraction, Number startValue, Number endValue) {
        float startFloat = startValue.floatValue();
        return startFloat + fraction * (endValue.floatValue() - startFloat);
    }
}
```

借助于 TypeEvaluator，属性动画就可以通过 ofObject() 来对不限定类型的属性做动画了：

1. 为目标属性写一个自定义的 TypeEvaluator
2. 使用 ofObject(Object target, String propertyName, TypeEvaluator evaluator, Object... values) 来创建 Animator

###### PropertyValuesHolder 同一个动画中改变多个属性

在同一个动画中会需要改变多个属性，需要使用 PropertyValuesHolder 

```java
PropertyValuesHolder holder1 = PropertyValuesHolder.ofFloat("scaleX", 1);  
PropertyValuesHolder holder2 = PropertyValuesHolder.ofFloat("scaleY", 1);  
PropertyValuesHolder holder3 = PropertyValuesHolder.ofFloat("alpha", 1);

ObjectAnimator animator = ObjectAnimator.ofPropertyValuesHolder(view, holder1, holder2, holder3)  animator.start();  
```

###### AnimatorSet 多个动画配合

如果在一个动画中改变多个属性，还要多个动画配合工作，就需要使用 AnimatorSet

```java
ObjectAnimator animator1 = ObjectAnimator.ofFloat(...);  
animator1.setInterpolator(new LinearInterpolator());  
ObjectAnimator animator2 = ObjectAnimator.ofInt(...);  
animator2.setInterpolator(new DecelerateInterpolator());

AnimatorSet animatorSet = new AnimatorSet();  
// 两个动画依次执行
animatorSet.playSequentially(animator1, animator2);  
// animator1 和 animator2 一起执行
animatorSet.play(animator1).with(animator2);  
// animator1 在 animator2 前执行
animatorSet.play(animator1).before(animator2);
// animator1 在 animator2 后执行
animatorSet.play(animator1).after(animator2); 
animatorSet.start();  
```

###### PropertyValuesHolders.ofKeyframe() 把同一个属性拆分

除了合并多个属性和调配多个动画，还可以在 PropertyValuesHolder 的基础上更进一步，通过设置  Keyframe （关键帧），把同一个动画属性拆分成多个阶段。

```java
// 在 0% 处开始
Keyframe keyframe1 = Keyframe.ofFloat(0, 0);  
// 时间经过 50% 的时候，动画完成度 100%
Keyframe keyframe2 = Keyframe.ofFloat(0.5f, 100);  
// 时间见过 100% 的时候，动画完成度倒退到 80%，即反弹 20%
Keyframe keyframe3 = Keyframe.ofFloat(1, 80);  
PropertyValuesHolder holder = PropertyValuesHolder.ofKeyframe("progress", keyframe1, keyframe2, keyframe3);
ObjectAnimator animator = ObjectAnimator.ofPropertyValuesHolder(view, holder);  
animator.start(); 
```

