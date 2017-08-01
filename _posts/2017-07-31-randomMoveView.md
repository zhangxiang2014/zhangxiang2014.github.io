---
layout: post
title:  android实现随机游走+触摸反馈以及跟随效果自定义view
date:   2017-07-31 20:13
categories: android
permalink: /archivers/randomMoveView
--

**我们先来看一段demo实现的效果:**


![随机游走动画](https://github.com/zhangxiang2014/zhangxiang2014.github.io/blob/master/_posts/randomView.gif)


gif动画分成3个部分：

- 随机游走
	>这部分的实现使用的是android属性动画，关于属性动画的基础知识这里不再详述，给一篇写的不错的文章[Android动画解析](http://www.jianshu.com/p/551f84402752)，大家可以跳过其他动画先看属性动画。我的实现里面并没有用到属性动画里面的所有属性，只用到了`scaleX`,`scaleY`,`translationX`,`translationY`,大家可以根据自己的需要扩展我的`list`(我将这些`ObjectAnimator`存成了数组。随机游走的原理实际上是数学上很简单的公式，这里我们加上递归的调用就很轻松的实现了随机游走，下面我会详述。

- 触摸反馈
	>这部分在gif里面的表现就是图片在一段时间内仅仅做放大缩小动画，实际的操作是我的手指触摸到了图片并保持按下动作，对应到代码里面我们会重写view的`dispatchTouchEvent`方法，在`ACTION_DOWN`动作里面停止之前的动画并重新开始一段动画。
	
- 跟随效果 
   >gif里面是后面移动很快那一段，实际上跟随的同时还在做**触摸反馈**里面提到的动画，实现跟随finger移动效果网上的实现有很多种，有复写`ondraw`方法的，有重写布局的...这里我们使用`view.scrollBy(int,int)`方法，实际上我也尝试使用了`view.scrollTo(int,int)`方法，不过效果不好，两者的差别可以参考[android 布局之滑动探究 scrollTo 和 scrollBy 方法使用说明](http://blog.csdn.net/vipzjyno1/article/details/24577023)这篇文章。
   
实际上还有一个部分，就是当手指离开屏幕以后view仍然会做随机游走动画，下面我就从上面三个部分辅以源码说明。
#控制器
为了方便操作view的属性动画,我在自定义view的里面加了一个内部类`AnimationControl`,外部类会初始化这个控制器并将自己传入到这个控制器中，以后所有对view操作的属性动画全部通过这个`AnimationControl `完成，代码如下：

    
```java
class AnimationControl {

        private View mView;
        private int angle;
        //只用到4个属性
        private ObjectAnimator[] animatorList = new ObjectAnimator[4];
        private AnimatorSet animationSet;
        public Boolean stop = true;

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

#全部源码




