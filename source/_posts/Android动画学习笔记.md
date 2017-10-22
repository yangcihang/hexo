---
title: Android动画学习笔记
date: 2017-10-21 23:15:28
tags: Android
categories: Android
---

动画效果一直是人机交互的一个非常重要的部分，动画效果的引入，会让交互变得更加友好，让用户获得更加愉悦的体验。作为一个前端开发者，动画也是一个必会的技能。<!--more-->

## View Animation

View Animation包含了Tween Animation、Frame Animation（Drawable Animation）。这些都是在安卓3.0之前的两种动画。

### 帧动画

帧动画有时也叫Drawable动画，它允许你实现像播放幻灯片一样的效果，这种动画的实质其实是Drawable，所以这种动画的XML定义方式文件 **一般放在res/drawable/目录下。**帧动画的动画本质呢就是我们的视觉残留。

一般我们会先在Drawable下面将动画资源引用好，然后在代码中调用start()/stop()来开始或者停止播放动画。当然我们也可以在Java代码中创建逐帧动画，创建AnimationDrawable对象，然后调用 addFrame(Drawable frame,int duration)向动画中添加帧，接着调用start()和stop()而已~推荐是使用XML来定义动画。

具体的标签有：

- `<animation-list>`：必须是根节点，包含多个`< item>`元素。属性有android:oneshot true代表只执行一次，false循环执行。
- `< item>`:animation-list的子项，包含的属性有：
	- android:drawable 一个frame的Drawable资源。
	- android:duration 一个frame显示多长时间。

举个例子，我们在drawable中

```
< ?xml version="1.0" encoding="utf-8"?>

< animation-list xmlns:android="http://schemas.android.com/apk/res/android" android:oneshot="false">
    < item android:drawable="@color/black" android:duration="300"/>
    < item android:drawable="@color/white" android:duration="300"/>
< /animation-list>


```
然后在kotlin中

```
  img_test.apply {
            setBackgroundResource(R.drawable.test_frame)
            setOnClickListener {
                if ((background as AnimationDrawable).isRunning)
                    (background as AnimationDrawable).stop()
                else (background as AnimationDrawable).start()
            }
        }
```
这样当我们点击图片的时候就会自动播放在drawable中设置的资源了。
**注意，Animation的start方法不能在Activity的onCreate方法中调用，因为AnimationDrawable还未完全附着到Window上。**

### 补间动画

 Tween Animation(补间动画)只能应用于View对象，而且只支持一部分属性，如支持缩放旋转而不支持背景颜色的改变。而且对于Tween Animation，并不改变属性的值，它只是改变了View对象绘制的位置，而没有改变View对象本身，比如，你有一个Button，坐标（100,100），Width:100,Height:100，而你有一个动画使其移动（200，200），你会发现动画过程中触发按钮点击的区域仍是(100,100)-(200,200)。 

补间动画通过XML或Android代码定义，建议还是使用XML文件定义，因为它更具可读性、可重用性。这里说一下，补间动画的所有父类都是它：

```
public abstract class Animation implements Cloneable
```

这个注意和属性动画的父类区分。


java类名 | xml关键字 | 描述信息
----|------|----
AlphaAnimation | <alpha> 放置在res/anim/目录下  |渐变透明度动画效果
RotateAnimation | <rotate> 放置在res/anim/目录下  | 画面转移旋转动画效果
ScaleAnimation | <scale> 放置在res/anim/目录下  | 渐变尺寸伸缩动画效果
TranslateAnimation | <translate> 放置在res/anim/目录下 | 画面转换位置移动动画效果
AnimationSet | <set> 放置在res/anim/目录下 | 一个持有其它动画元素alpha、scale、translate、rotate或者其它set元素的容器

#### Animation属性
 
 xml属性 | java方法 | 解释
----|------|----
android:detachWallpaper |setDetachWallpaper(boolean)|是否在壁纸上运行
android:duration	| setDuration(long) |	动画持续时间，毫秒为单位
android:fillAfter| setFillAfter(boolean)	| 控件动画结束时是否保持动画最后的状态
android:fillBefore | setFillBefore(boolean) | 控件动画结束时是否还原到开始动画前的状态
android:fillEnabled | setFillEnabled(boolean) | 与android:fillBefore效果相同
android:interpolator | setInterpolator(Interpolator) | 设定插值器（指定的动画效果，譬如回弹等）
android:repeatCount | setRepeatCount(int) | 重复次数
android:repeatMode | setRepeatMode(int)	| 重复类型有两个值，reverse表示倒序回放，restart表示从头播放
android:startOffset | setStartOffset(long) | 调用start函数之后等待开始运行的时间，单位为毫秒
android:zAdjustment	| setZAdjustment(int) | 表示被设置动画的内容运行时在Z轴上的位置（top/bottom/normal），默认为normal

####  Alpha属性

 xml属性 | java方法 | 解释
 ----|------|----
android:fromAlpha | AlphaAnimation(float fromAlpha, …) | 动画开始的透明度（0.0到1.0，0.0是全透明，1.0是不透明）
android:toAlpha | AlphaAnimation(…, float toAlpha) | 动画结束的透明度，同上

#### Rotate属性

 xml属性 | java方法 | 解释
----|------|----
android:fromDegrees | RotateAnimation(float fromDegrees, …) | 旋转开始角度，正代表顺时针度数，负代表逆时针度数
android:toDegrees | RotateAnimation(…, float toDegrees, …) | 旋转结束角度，正代表顺时针度数，负代表逆时针度数
android:pivotX | RotateAnimation(…, float pivotX, …) | 缩放起点X坐标（数值、百分数、百分数p，譬如50表示以当前View左上角坐标加50px为初始点、50%表示以当前View的左上角加上当前View宽高的50%做为初始点、50%p表示以当前View的左上角加上父控件宽高的50%做为初始点）
android:pivotY | RotateAnimation(…, float pivotY) | 缩放起点Y坐标，同上规律

#### Scale属性

xml属性 | java方法 | 解释
----|------|----
android:fromXScale | ScaleAnimation(float fromX, …) | 初始X轴缩放比例，1.0表示无变化
android:toXScale	| ScaleAnimation(…, float toX, …) | 结束X轴缩放比例
android:fromYScale | ScaleAnimation(…, float fromY, …) | 初始Y轴缩放比例
android:toYScale | ScaleAnimation(…, float toY, …) | 结束Y轴缩放比例
android:pivotX | ScaleAnimation(…, float pivotX, …) | 缩放起点X轴坐标（数值、百分数、百分数p，譬如50表示以当前View左上角坐标加50px为初始点、50%表示以当前View的左上角加上当前View宽高的50%做为初始点、50%p表示以当前View的左上角加上父控件宽高的50%做为初始点）
android:pivotY | ScaleAnimation(…, float pivotY) | 缩放起点Y轴坐标，同上规律

#### Translate属性详解

xml属性 | java方法 | 解释
----|------|----
android:fromXDelta | TranslateAnimation(float fromXDelta, …) | 起始点X轴坐标（数值、百分数、百分数p，譬如50表示以当前View左上角坐标加50px为初始点、50%表示以当前View的左上角加上当前View宽高的50%做为初始点、50%p表示以当前View的左上角加上父控件宽高的50%做为初始点）
android:fromYDelta | TranslateAnimation(…, float fromYDelta, …) | 起始点Y轴从标，同上规律
android:toXDelta | TranslateAnimation(…, float toXDelta, …) | 结束点X轴坐标，同上规律
android:toYDelta	| TranslateAnimation(…, float toYDelta) | 结束点Y轴坐标，同上规律

#### 插值器

插值器是用来控制动画的变化速度，可以理解成动画渲染器，当然我们也可以自己实现Interpolator 接口，自行来控制动画的变化速度，而Android中已经为我们提供了五个可供选择的实现类:

- LinearInterpolator：动画以均匀的速度改变
- AccelerateInterpolator：在动画开始的地方改变速度较慢，然后开始加速
- AccelerateDecelerateInterpolator：在动画开始、结束的地方改变速度较慢，中间时加速
- CycleInterpolator：动画循环播放特定次数，变化速度按正弦曲线改变： Math.sin(2 * mCycles * Math.PI * input)
- DecelerateInterpolator：在动画开始的地方改变速度较快，然后开始减速
- AnticipateInterpolator：反向，先向相反方向改变一段再加速播放
- AnticipateOvershootInterpolator：开始的时候向后然后向前甩一定值后返回最后的值
- BounceInterpolator： 跳跃，快到目的值时值会跳跃，如目的值100，后面的值可能依次为85，77，70，80，90，100
- OvershottInterpolator：回弹，最后超出目的值然后缓慢改变到目的值

#### 使用简介：

上面这些都是补间动画的一些xml和java方法的简介，这些方法属性记不住不要紧，咱们可以随时查看的。最重要的还是怎么去用。其实用法也是特别简单的，十分的套路，我们只需要记住套路就可以。我们就举个简单的旋转动画的例子吧。

首先是XML

```
< ?xml version="1.0" encoding="utf-8"?>
< rotate xmlns:android="http://schemas.android.com/apk/res/android"
        android:interpolator="@android:anim/accelerate_decelerate_interpolator"
        android:fromDegrees="0"
        android:toDegrees="360"
        android:duration="1000"
        android:pivotX="50%"
        android:pivotY="50%"
        android:repeatCount="infinite"
    />
```
其中我们用了`@android:animaccelerate_decelerate_interpolator`这个插值器，就是上面所说的DecelerateInterpolator在动画开始的地方改变速度较快，然后开始减速。

写完XML后，在java代码中

```
	Animation animation = AnimationUtils.loadAnimation(this, R.anim.anim_rotate);
	testImg.setAnimation(animation);
	animation.start();
	animation.cancel();
```
我们可以看到，先创建了一个Animation对象，然后在用AnimationUtils的loadAnimation方法将XML里定义的动画加载到animation对象中，然后通过View的setAnimation方法将animation传入，这样就将这个动画绑定到view上面了。最后通过animation的start和cancel方法来开始或者取消动画。  其中我们可以为animation设置动画的监听，这个特别简单就不再这赘述了...

## Property Animation
Android 3.0以后引入了属性动画，属性动画可以轻而易举的实现许多View动画做不到的事。其实说白了，记住一点就行，属性动画实现原理就是修改控件的属性值实现的动画。当然，功能强大的代价就是使用起来要比较复杂。

属性动画所有的父类是它：

```
public abstract class Animator implements Cloneable 
```
这个注意要和补间动画来区分。

接下来介绍一下相关的API：

- Animator  创建属性动画的基类，一般不直接用，而是用他的两个子类
- ValueAnimator  Animator的直接派生类。其内部采用一种时间循环的机制来计算值与值之间的动画过度，我们只需将初始值以及结束值提供给该类，并告诉其动画所需时间长度，该类就会自动帮我们从初始值平滑过度到结束。该类还能管理动画的播放次数、模式和监听器等。
- AnimatorSet  Animator的直接派生类，可以组合多个Animator，并制定Animator的播放次序。
- ObjectAnimator  ValueAnimator的子类，允许我们对指定对象的属性执行动画，用起来更简单，实际中用得较多。
- Evaluator  计算器，告诉动画系统如何从初始值过度到结束值。提供了一下的几种Evaluator：
	- IntEvaluator:用于计算int属性
	- FloatEvaluator:用于计算float属性
	- ArgbEvaluator：用于计算16进制表示颜色值的计算器
	- TypeEvaluator：上述计算类的公共接口，可以自己实现接口完成自定义。

### ValueAnimator
使用流程：

1. 调用ValueAnimator的ofInt()，ofFloat()或ofObject()静态方法创建ValueAnimator实例
2. 调用实例的setXxx方法设置动画持续时间，插值方式，重复次数等
3. 调用实例的addUpdateListener添加AnimatorUpdateListener监听器，在该监听器中 可以获得ValueAnimator计算出来的值，你可以值应用到指定对象上~
4. 调用实例的start()方法开启动画！ 另外我们可以看到ofInt和ofFloat都有个这样的参数：float/int... values代表可以多个值！

举个例子:

```
  //按轨迹方程来运动
    private void lineAnimator() {
        width = ly_root.getWidth();
        height = ly_root.getHeight();
        ValueAnimator xValue = ValueAnimator.ofInt(height,0,height / 4,height / 2,height / 4 * 3 ,height);
        xValue.setDuration(3000L);
        xValue.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
            @Override
            public void onAnimationUpdate(ValueAnimator animation) {
                // 轨迹方程 x = width / 2
                int y = (Integer) animation.getAnimatedValue();
                int x = width / 2;
                moveView(img_babi, x, y);
            }
        });
        xValue.setInterpolator(new LinearInterpolator());
        xValue.start();
    }
        
        
    private void moveView(View view, int rawX, int rawY) {
        int left = rawX - img_babi.getWidth() / 2;
        int top = rawY - img_babi.getHeight();
        int width = left + view.getWidth();
        int height = top + view.getHeight();
        view.layout(left, top, width, height);
    }
```

其中moveView方法是将View重新布局，xValue的值从参数列表就可以看出，然后设置每次更新的监听，在监听中每次调用moveView方法来改变View的布局。如果是组合动画的话，我们可以这样：

```
//缩放效果
    private void scaleAnimator(){
    
        final float scale = 0.5f;
        AnimatorSet scaleSet = new AnimatorSet();
        ValueAnimator valueAnimatorSmall = ValueAnimator.ofFloat(1.0f, scale);
        valueAnimatorSmall.setDuration(500);

        ValueAnimator valueAnimatorLarge = ValueAnimator.ofFloat(scale, 1.0f);
        valueAnimatorLarge.setDuration(500);

        valueAnimatorSmall.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
            @Override
            public void onAnimationUpdate(ValueAnimator animation) {
                float scale = (Float) animation.getAnimatedValue();
                img_babi.setScaleX(scale);
                img_babi.setScaleY(scale);
            }
        });
        valueAnimatorLarge.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
            @Override
            public void onAnimationUpdate(ValueAnimator animation) {
                float scale = (Float) animation.getAnimatedValue();
                img_babi.setScaleX(scale);
                img_babi.setScaleY(scale);
            }
        });

        scaleSet.play(valueAnimatorLarge).after(valueAnimatorSmall);
        scaleSet.start();
     }
```

这个组合动画我们可以用play after来设定动画实例，然后动画就先执行play的动画再执行after的实例了。当然，类似的方法还有一个with，就是两个动画会一起播放。

- after(Animator anim)   将现有动画插入到传入的动画之后执行
- after(long delay)   将现有动画延迟指定毫秒后执行
- before(Animator anim)   将现有动画插入到传入的动画之前执行
- with(Animator anim)   将现有动画和传入的动画同时执行

### ObjectAnimator
相比于ValueAnimator，ObjectAnimator可能才是我们最常接触到的类，因为ValueAnimator只不过是对值进行了一个平滑的动画过渡，但我们实际使用到这种功能的场景好像并不多。而ObjectAnimator则就不同了，它是可以直接对任意对象的任意属性进行动画操作的，比如说View的alpha属性。还有ObjectAnimator在设计的时候就没有针对于View来进行设计，而是针对于任意对象的。

既然ObjectAnimator是继承自ValueAnimator的，那么它的用法应该是和ValueAnimator相似的。

```
 ObjectAnimator.ofFloat(textview, "alpha", 1f, 0f);
```
其实这段代码的意思就是ObjectAnimator会帮我们不断地改变textview对象中alpha属性的值，从1f变化到0f。然后textview对象需要根据alpha属性值的改变来不断刷新界面的显示，从而让用户可以看出淡入淡出的动画效果。

那么textview对象中是不是有alpha属性这个值呢？没有，不仅textview没有这个属性，连它所有的父类也是没有这个属性的！这就奇怪了，textview当中并没有alpha这个属性，ObjectAnimator是如何进行操作的呢？其实ObjectAnimator内部的工作机制并不是直接对我们传入的属性名进行操作的，而是会去寻找这个属性名对应的get和set方法，因此alpha属性所对应的get和set方法应该就是：

```
public void setAlpha(float value);  
public float getAlpha();  
```

ObjectAnimator的使用相比ValueAnimator简单多了，我们只需要将它实例化后，其他的用法和ValueAnimator是一样的。

### 动画的监听
接下来要说下动画事件的监听，上面我们ValueAnimator的监听器是 AnimatorUpdateListener，当值状态发生改变时候会回调onAnimationUpdate方法！
除了这种事件外还有：动画进行状态的监听, AnimatorListener，我们可以调用addListener方法 添加监听器，然后重写下面四个回调方法：

- onAnimationStart()：动画开始
- onAnimationRepeat()：动画重复执行
- onAnimationEnd()：动画结束
- onAnimationCancel()：动画取消

加入AnimatorListener的话，四个方法你都要重写，这样写起来很是麻烦，不过Android已经给我们提供好一个适配器类：AnimatorListenerAdapter，该类中已经把每个接口 方法都实现好了，所以我们这里只写一个回调方法也是可以的。

## Evaluator
我们在调用属性动画的时候，用到了ofInt,ofFloat,ofObject这些静态方法来创建ValueAnimator的实例，在例子中，我们用到了ofInt,ofFloat，但是细心的同学可能发现了，ValueAnimator还有个ofObject方法来构造ValueAnimator实例，那么这个是干什么用的呢？

```

    public static ValueAnimator ofObject(TypeEvaluator evaluator, Object... values) {
        ValueAnimator anim = new ValueAnimator();
        anim.setObjectValues(values);
        anim.setEvaluator(evaluator);
        return anim;
    }
```
我们先戳进去看看这个源码的参数。第一个参数是名为TypeEvaluator的一个东西，这就是我们要说的计算器，他是来告诉动画系统如何从初始值过渡到结束值的。前面提到过，TypeEvaluator是所有Evaluator的基类，这里再重复一遍，系统提供了以下的几种Evaluator:

- IntEvaluator:用于计算int属性
- FloatEvaluator:用于计算float属性
- ArgbEvaluator：用于计算16进制表示颜色值的计算器
- TypeEvaluator：上述计算类的公共接口，可以自己实现接口完成自定义。

当然，这些Evaluator都是TypeEvaluator实现。我们先来看看TypeEvaluator这个接口长什么样吧：

```
public interface TypeEvaluator<T> {
    public T evaluate(float fraction, T startValue, T endValue);

}
```

就这个一个evaluate方法，简单吧？这里面的三个参数的含义依次是：

- fraction：动画的完成度，我们根据他来计算动画的值应该是多少
- startValue：动画的起始值
- endValue：动画的结束值

那么我们来看看系统是如何实现TypeEvaluator这个接口的，咱们就从IntEvaluator说起：

```
public class IntEvaluator implements TypeEvaluator<Integer> {
    public Integer evaluate(float fraction, Integer startValue, Integer endValue) {
        int startInt = startValue;
        return (int)(startInt + fraction * (endValue - startInt));
    }
}
```

实现了TypeEvaluator接口，重写了evaluate方法，其中返回的是动画的值。动画的值是多少呢？我们从IntEvaluator的return语句中可以看出，**动画的值 = 初始值 + 完成度 * (结束值 - 初始值)** 这样当完成度为100%的时候，动画的值就是结束值了，同样的还有FloatEvaluator等。细心的同学也发现了，这个TypeEvaluator里面传入的是一个泛型，这个类型也就是我们接下来自定义的Object.

现在我们举个自定义Evaluator的例子。

```
package com.example.yang.testkotlin.widget;

import android.animation.AnimatorSet;
import android.animation.ObjectAnimator;
import android.animation.TypeEvaluator;
import android.animation.ValueAnimator;
import android.content.Context;
import android.graphics.Canvas;
import android.graphics.Color;
import android.graphics.Paint;
import android.graphics.Point;
import android.support.annotation.Nullable;
import android.util.AttributeSet;
import android.view.View;

/**
 * @author YangCihang
 * @since 17/10/18.
 * email yangcihang@hrsoft.net
 */

public class CircleAnimView extends View {
    public static final int RADIUS = 80;
    private Point currentPoint;
    private Paint mPaint;
    private int mColor;//必须写这个属性的get和set，否则动画无法识别color属性

    public CircleAnimView(Context context) {
        super(context);
    }

    public CircleAnimView(Context context, @Nullable AttributeSet attrs) {
        super(context, attrs);
        init();
    }

    public CircleAnimView(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
    }

    private void init() {
        mPaint = new Paint(Paint.ANTI_ALIAS_FLAG);
        mPaint.setColor(Color.BLUE);
    }

    @Override
    protected void onDraw(Canvas canvas) {
        if (currentPoint == null) {
            currentPoint = new Point(RADIUS, RADIUS);
            drawCircle(canvas);
            startAnimation();
        } else {
            drawCircle(canvas);
        }
    }

    private void drawCircle(Canvas canvas) {
        float x = currentPoint.x;
        float y = currentPoint.y;
        //用canvas draw，因此此控件大小应该为全屏大小才可以从左上到右下
        canvas.drawCircle(x, y, RADIUS, mPaint);
    }

    private void startAnimation() {
        Point startPoint = new Point(RADIUS, RADIUS);
        Point endPoint = new Point(getWidth() - RADIUS, getHeight() - RADIUS);
        ValueAnimator anim = ValueAnimator.ofObject(new PointEvaluator(), startPoint, endPoint);
        anim.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
            @Override
            public void onAnimationUpdate(ValueAnimator animation) {
                currentPoint = (Point) animation.getAnimatedValue();
                invalidate();
            }
        });

        ObjectAnimator objectAnimator = ObjectAnimator.ofObject(this, "color", new ColorEvaluator(),
                Color.BLUE, Color.RED);
        //动画集合将两个动画加到一起，with同时播放
        AnimatorSet animatorSet = new AnimatorSet();
        animatorSet.play(anim).with(objectAnimator);
        animatorSet.setStartDelay(1000L);
        animatorSet.setDuration(3000L);
        animatorSet.start();
    }

    /**
     * 坐标变化
     */
    class PointEvaluator implements TypeEvaluator<Point> {

        @Override
        public Point evaluate(float fraction, Point startValue, Point endValue) {
            //初始值 + 完成度 * (结束值 - 初始值)
            int x = (int) (startValue.x + fraction * (endValue.x - startValue.x));
            int y = (int) (startValue.y + fraction * (endValue.y - startValue.y));
            return new Point(x, y);
        }
    }

    /**
     * 颜色变化
     */
    public class ColorEvaluator implements TypeEvaluator<Integer> {
        @Override
        public Integer evaluate(float fraction, Integer startValue, Integer endValue) {
            int alpha = (int) (Color.alpha(startValue) + fraction *
                    (Color.alpha(endValue) - Color.alpha(startValue)));
            int red = (int) (Color.red(startValue) + fraction *
                    (Color.red(endValue) - Color.red(startValue)));
            int green = (int) (Color.green(startValue) + fraction *
                    (Color.green(endValue) - Color.green(startValue)));
            int blue = (int) (Color.blue(startValue) + fraction *
                    (Color.blue(endValue) - Color.blue(startValue)));
            return Color.argb(alpha, red, green, blue);
        }
    }

    //color的get和set方法~
    public int getColor() {
        return mColor;
    }

    public void setColor(int color) {
        mColor = color;
        mPaint.setColor(color);
        invalidate();
    }
}

```

在这里我们定义了两个Evaluator类分别来表示坐标变化和颜色变化。注意，我们要改变color属性的时候，一定要写setColor和getColor方法，原因上面已经说过了。

## Interpolator

Interpolator中文译名叫做插值器或者补间器。在补间动画的时候我们介绍了几个常用的插值器，我们可以回头去看看。补间动画和属性动画都可以用插值器的。而且补间动画还新增加了一个TimeInterpolator接口，该接口是用于兼容之前的Interpolator的(也就是Interpolator又继承了TimeInterpolator接口)，这使得所有过去的Interpolator实现类都可以直接拿过来 放到属性动画当中使用！我们可以调用动画对象的setInterpolator()方法设置不同的Interpolator。比如:

```
animatorSet.setInterpolator(new AccelerateInterpolator(2f))
```
括号里面的值用于控制加速度的。

### 自定义Interpolator
我们自定义Interpolator其实就只要实现TimeInterpolator就可以了，重写也就只需要重写getInterpolation方法比如:

```

private class DecelerateAccelerateInterpolator implements TimeInterpolator {
    @Override
    public float getInterpolation(float input) {
        if (input < 0.5) {
            return (float) (Math.sin(input * Math.PI) / 2);
        } else {
            return 1 - (float) (Math.sin(input * Math.PI) / 2);
        }
    }
}
```

也是很简单的嘛。不过我们如果需要做一些复杂的速率变化，就得需要数学功底了。
