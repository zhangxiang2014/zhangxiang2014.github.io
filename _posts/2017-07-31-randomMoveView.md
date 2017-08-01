---
layout: post
title:  android实现随机游走+触摸反馈+跟随效果的自定义view
date:   2017-07-31 20:13
categories: android
permalink: /archivers/randomMoveView
---

**我们先来看一段demo实现的效果:**


![随机游走动画](https://github.com/zhangxiang2014/zhangxiang2014.github.io/blob/master/_posts/randomView.gif?raw=true)


gif动画分成3个部分：

- 随机游走
	>这部分的实现使用的是android属性动画，关于属性动画的基础知识这里不再详述，给一篇写的不错的文章[Android动画解析](http://www.jianshu.com/p/551f84402752)，大家可以跳过其他动画先看属性动画。我的实现里面并没有用到属性动画里面的所有属性，只用到了`scaleX`,`scaleY`,`translationX`,`translationY`,大家可以根据自己的需要扩展我的`list`(我将这些`ObjectAnimator`存成了数组)。随机游走的原理是数学上很简单的公式，这里我们加上递归的调用就很轻松的实现了，下面我会详述。

- 触摸反馈
	>这部分在gif里面的表现就是图片在一段时间内仅仅做放大缩小动画，实际的操作是我的手指触摸到了图片并保持按下动作，对应到代码里面我们会重写view的`dispatchTouchEvent`方法，在`ACTION_DOWN`动作里面停止之前的动画并重新开始一段动画。
	
- 跟随效果 
   >gif里面是后面移动很快那一段，实际上跟随的同时还在做**触摸反馈**里面提到的动画，实现跟随finger移动效果网上的实现有很多种，有复写`ondraw`方法的，有重写布局的...这里我们使用`view.scrollBy(int,int)`方法，实际上我也尝试使用了`view.scrollTo(int,int)`方法，不过效果不好，两者的差别可以参考[android 布局之滑动探究 scrollTo 和 scrollBy 方法使用说明](http://blog.csdn.net/vipzjyno1/article/details/24577023)这篇文章。
   
实际上还有一个部分，就是当手指离开屏幕以后view仍然会做随机游走动画，下面我就从上面三个部分辅以源码说明。
# 控制器

为了方便操作view的属性动画,我在自定义view的里面加了一个内部类`AnimationControl`,外部类会初始化这个控制器并将自己传入到这个控制器中，以后所有对view操作的属性动画全部通过这个`AnimationControl `完成，代码如下：

    
```java
class AnimationControl {

        private View mView;
        //角度
        private int angle;
        //只用到4个属性
        private ObjectAnimator[] animatorList = new ObjectAnimator[4];
        private AnimatorSet animationSet;
        public Boolean stop = true;
        //默认值
        private long defaultMoveDistance = 300;
        private long defaultScale = 2;
        private long defaultDurtion = 3000;

        public AnimationControl(View mView) {
            this.mView = mView;
        }

        //此方法中实现了随机游走，并根据传入参数决定进行随机游走动画还是触摸反馈动画
        public void initDefaultAnimation(final Boolean isOpposite) {
            long scale = defaultScale;
            long durtion = defaultDurtion;
            animationSet = new AnimatorSet();
            if (isOpposite) {
                scale += defaultScale;
            } else {
                angle = new Random().nextInt(360);
            }
            double moveX = defaultMoveDistance  * Math.cos(angle);
            double moveY = defaultMoveDistance * Math.sin(angle);
            animatorList[0] = animatorList[0].ofFloat(mView,"scaleX", 1f, scale, 1f);
            animatorList[1] = animatorList[1].ofFloat(mView,"scaleY", 1f, scale, 1f);
            animatorList[2] = animatorList[2].ofFloat(mView,"translationX", mView.getTranslationX(), (float) moveX );
            animatorList[3] = animatorList[3].ofFloat(mView,"translationY", mView.getTranslationY(), (float) moveY );
            if (isOpposite) {
                durtion = defaultDurtion/2;
                animationSet.setDuration(durtion);
            } else {
                animationSet.setDuration(durtion);
            }
            animationSet.setInterpolator(new AccelerateDecelerateInterpolator());
            if (isOpposite) {
                animationSet.playTogether(animatorList[0],animatorList[1]);
            } else {
                animationSet.playTogether(animatorList);
            }
            animationSet.addListener(new Animator.AnimatorListener() {
                @Override
                public void onAnimationStart(Animator animation) {

                }

                @Override
                public void onAnimationEnd(Animator animation) {
                    //在监听器中监听动画是否停止，如果没有停止递归调用startAnmiation方法开始下一次游走的动画
                    if(stop) {
                        return;
                    } else {
                        startAnmiation(isOpposite);
                    }
                }

                @Override
                public void onAnimationCancel(Animator animation) {

                }

                @Override
                public void onAnimationRepeat(Animator animation) {

                }
            });
        }
        //开始动画
        public void startAnmiation(Boolean isOpposite) {
            stop = false;
            initDefaultAnimation(isOpposite);
            animationSet.start();
        }
        //为动画设置属性，这里传入的参数是随机游走的范围和放大的倍数，如有需要，可以扩展这些参数
        public void setAnmiationPropoty(long distance,long scale){
            defaultMoveDistance = distance;
            defaultScale = scale;
        }
        //停止动画
        public void stopAnmiation() {
            stop = true;
            animationSet.cancel();
        }

        public void stopBeforeAndOppositeMove() {
            stopAnmiation();
            startAnmiation(true);
        }

    }
```

<<<<<<< HEAD
# 随机游走原理

=======
#随机游走原理
>>>>>>> 9c630ce4e0c3f6873100c8d7e30d9a3ed83ed88b
实际上原理很简单，我们只要把每次游走的路径变成随机的就可以，如何定义随机呢？由于是2D平面，我们只需要随机出一个角度x，并定义游走的范围L（实际上也可以用随机数来初始化一个半径，这里我们用给定的值），那么在x轴方向上的移动的距离就是L* cosX,y轴则为L *sinX,剩下的就交给属性动画和递归就可以了。代码如下：


```java
	        //此方法中实现了随机游走，并根据传入参数决定进行随机游走动画还是触摸反馈动画
        public void initDefaultAnimation(final Boolean isOpposite) {
            long scale = defaultScale;
            long durtion = defaultDurtion;
            animationSet = new AnimatorSet();
            if (isOpposite) {
                scale += defaultScale;
            } else {
                //随机出一个角度
                angle = new Random().nextInt(360);
            }
            //在x,y轴上分别移动的距离
            double moveX = defaultMoveDistance  * Math.cos(angle);
            double moveY = defaultMoveDistance * Math.sin(angle);
            animatorList[0] = animatorList[0].ofFloat(mView,"scaleX", 1f, scale, 1f);
            animatorList[1] = animatorList[1].ofFloat(mView,"scaleY", 1f, scale, 1f);
            animatorList[2] = animatorList[2].ofFloat(mView,"translationX", mView.getTranslationX(), (float) moveX );
            animatorList[3] = animatorList[3].ofFloat(mView,"translationY", mView.getTranslationY(), (float) moveY );
            if (isOpposite) {
                durtion = defaultDurtion/2;
                animationSet.setDuration(durtion);
            } else {
                animationSet.setDuration(durtion);
            }
            //设置加速度控制器
            animationSet.setInterpolator(new AccelerateDecelerateInterpolator());
            if (isOpposite) {
                animationSet.playTogether(animatorList[0],animatorList[1]);
            } else {
                animationSet.playTogether(animatorList);
            }
            animationSet.addListener(new Animator.AnimatorListener() {
                @Override
                public void onAnimationStart(Animator animation) {

                }

                @Override
                public void onAnimationEnd(Animator animation) {
                    //在监听器中监听动画是否停止，如果没有停止递归调用startAnmiation方法开始下一次游走的动画
                    //每次动画start之前一定会再次初始化属性
                    if(stop) {
                        return;
                    } else {
                        startAnmiation(isOpposite);
                    }
                }

                @Override
                public void onAnimationCancel(Animator animation) {

                }

                @Override
                public void onAnimationRepeat(Animator animation) {

                }
            });
        }
```

<<<<<<< HEAD
# 触摸反馈+跟随效果

=======
#触摸反馈+跟随效果
>>>>>>> 9c630ce4e0c3f6873100c8d7e30d9a3ed83ed88b
先看源码


```java
@Override
    public boolean dispatchTouchEvent(MotionEvent event) {
        final int X = (int) event.getRawX();
        final int Y = (int) event.getRawY();
        if (!mAnimationControl.stop) {
            switch (event.getAction()) {
                case MotionEvent.ACTION_DOWN:
                    lastX = X;
                    lastY = Y;
                    isOpposite = true;
                    mAnimationControl.stopBeforeAndOppositeMove();
                    break;
                case MotionEvent.ACTION_MOVE:
                    if (isOpposite) {
                        //计算移动的距离
                        int offX = X - lastX;
                        int offY = Y - lastY;
                        ((View)getParent()).scrollBy(-offX,-offY);
                        lastX = X;
                        lastY = Y;
                    }
                    break;
                case MotionEvent.ACTION_UP:
                case MotionEvent.ACTION_CANCEL:
                    if (isOpposite) {
                        isOpposite = false;
                        mAnimationControl.stopAnmiation();
                        mAnimationControl.startAnmiation(false);
                    }
                    break;

            }
        }
        return true;
    }
```
可以看到只有当动画处在非stop的情况下才有**触摸反馈**和**跟随动画**    
1.触摸反馈      
 >只需要关心`MotionEvent.ACTION_DOWN`,这里面我们调用了控制器的`mAnimationControl.stopBeforeAndOppositeMove()`方法开始了一段新的动画
 
2.跟随动画
>我们需要记录上次移动的位置并计算出移动的偏移量，这是因为`scrollBy`方法是根据当前位置移动的，同时我们在`MotionEvent.ACTION_UP`,方法中又恢复了随机游走动画。

<<<<<<< HEAD
# 自定义view全部源码

=======
#自定义view全部源码
>>>>>>> 9c630ce4e0c3f6873100c8d7e30d9a3ed83ed88b
这个自定义view完全可以当作一个普通的view来用，只不过你可以通过`startRandomMove()`和`stopRandomMove()`来启动和停止动画，动画里面可以做的事情很多，可以根据自己的需求定制动画和触摸反馈。

```java
package com.example.mac.myapplication;

import android.animation.Animator;
import android.animation.AnimatorSet;
import android.animation.ObjectAnimator;
import android.content.Context;
import android.support.annotation.Nullable;
import android.util.AttributeSet;
import android.view.MotionEvent;
import android.view.View;
import android.view.animation.AccelerateDecelerateInterpolator;
import java.util.Random;

/**
 * Created by zhangxiang on 2017/7/16.
 */

public class RandomMoveView extends View {

    private Context mContext;
    private AnimationControl mAnimationControl;
    private Boolean isOpposite;
    private Boolean isStop = true;
    private int lastX, lastY;

    public RandomMoveView(Context context) {
        this(context,null);
    }

    public RandomMoveView(Context context, @Nullable AttributeSet attrs) {
        super(context,attrs);
        mContext = context;
        mAnimationControl = new AnimationControl(this);
    }

    public void startRandomMove() {
        if (isStop) {
            mAnimationControl.startAnmiation(false);
            isStop = false;
        }
    }

    public void stopRandomMove() {
        if (!isStop) {
            mAnimationControl.stopAnmiation();
            isStop = true;
        }
    }

    public void setAnmiationPropoty(long distance,long scale) {
        mAnimationControl.setAnmiationPropoty(distance,scale);
    }

    @Override
    public boolean dispatchTouchEvent(MotionEvent event) {
        final int X = (int) event.getRawX();
        final int Y = (int) event.getRawY();
        if (!mAnimationControl.stop) {
            switch (event.getAction()) {
                case MotionEvent.ACTION_DOWN:
                    lastX = X;
                    lastY = Y;
                    isOpposite = true;
                    mAnimationControl.stopBeforeAndOppositeMove();
                    break;
                case MotionEvent.ACTION_MOVE:
                    if (isOpposite) {
                        //计算移动的距离
                        int offX = X - lastX;
                        int offY = Y - lastY;
                        ((View)getParent()).scrollBy(-offX,-offY);
                        lastX = X;
                        lastY = Y;
                    }
                    break;
                case MotionEvent.ACTION_UP:
                case MotionEvent.ACTION_CANCEL:
                    if (isOpposite) {
                        isOpposite = false;
                        mAnimationControl.stopAnmiation();
                        mAnimationControl.startAnmiation(false);
                    }
                    break;

            }
        }
        return true;
    }

    class AnimationControl {

        private View mView;
        //角度
        private int angle;
        //只用到4个属性
        private ObjectAnimator[] animatorList = new ObjectAnimator[4];
        private AnimatorSet animationSet;
        public Boolean stop = true;
        //默认值
        private long defaultMoveDistance = 300;
        private long defaultScale = 2;
        private long defaultDurtion = 3000;

        public AnimationControl(View mView) {
            this.mView = mView;
        }

        //此方法中实现了随机游走，并根据传入参数决定进行随机游走动画还是触摸反馈动画
        public void initDefaultAnimation(final Boolean isOpposite) {
            long scale = defaultScale;
            long durtion = defaultDurtion;
            animationSet = new AnimatorSet();
            if (isOpposite) {
                scale += defaultScale;
            } else {
                //随机出一个角度
                angle = new Random().nextInt(360);
            }
            //在x,y轴上分别移动的距离
            double moveX = defaultMoveDistance  * Math.cos(angle);
            double moveY = defaultMoveDistance * Math.sin(angle);
            animatorList[0] = animatorList[0].ofFloat(mView,"scaleX", 1f, scale, 1f);
            animatorList[1] = animatorList[1].ofFloat(mView,"scaleY", 1f, scale, 1f);
            animatorList[2] = animatorList[2].ofFloat(mView,"translationX", mView.getTranslationX(), (float) moveX );
            animatorList[3] = animatorList[3].ofFloat(mView,"translationY", mView.getTranslationY(), (float) moveY );
            if (isOpposite) {
                durtion = defaultDurtion/2;
                animationSet.setDuration(durtion);
            } else {
                animationSet.setDuration(durtion);
            }
            //设置加速度控制器
            animationSet.setInterpolator(new AccelerateDecelerateInterpolator());
            if (isOpposite) {
                animationSet.playTogether(animatorList[0],animatorList[1]);
            } else {
                animationSet.playTogether(animatorList);
            }
            animationSet.addListener(new Animator.AnimatorListener() {
                @Override
                public void onAnimationStart(Animator animation) {

                }

                @Override
                public void onAnimationEnd(Animator animation) {
                    //在监听器中监听动画是否停止，如果没有停止递归调用startAnmiation方法开始下一次游走的动画
                    //每次动画start之前一定会再次初始化属性
                    if(stop) {
                        return;
                    } else {
                        startAnmiation(isOpposite);
                    }
                }

                @Override
                public void onAnimationCancel(Animator animation) {

                }

                @Override
                public void onAnimationRepeat(Animator animation) {

                }
            });
        }
        //开始动画
        public void startAnmiation(Boolean isOpposite) {
            stop = false;
            initDefaultAnimation(isOpposite);
            animationSet.start();
        }
        //为动画设置属性，这里传入的参数是随机游走的范围和放大的倍数，如有需要，可以扩展这些参数
        public void setAnmiationPropoty(long distance,long scale){
            defaultMoveDistance = distance;
            defaultScale = scale;
        }
        //停止动画
        public void stopAnmiation() {
            stop = true;
            animationSet.cancel();
        }

        public void stopBeforeAndOppositeMove() {
            stopAnmiation();
            startAnmiation(true);
        }

    }
}
```



