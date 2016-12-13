---
title: Android自定义View实战之角度选择器
date: 2016-11-5 14:46:45
tags: [Android]
---

---

本文比较基础，在阅读本文前只需要掌握最基础的自定义View知识和Android事件知识。

###	起步

有一天晚上，在Google Photos查看照片，用了一下它的图片剪裁功能，于是我马上就被其界面和操作吸引。

![](http://7xrqmj.com1.z0.glb.clouddn.com/gpcrop.png?imageView2/2/w/360)



第二天我就想模仿做一个这样的裁图库，当然，我做了。同时也做了一个和Google Photos裁图页面几乎一模一样的角度选择器。那么，来看一下最终的效果：

![](http://7xrqmj.com1.z0.glb.clouddn.com/DegreeSeekbar.gif)

### 思路

仔细观察这个效果，先分析构成结构，我把它分成三部分：

1. 表示刻度的点
2. 相应点上方的数字
3. 控件中央的当前刻度与三角

可以看出，构成元素十分简单，不涉及图片，`Drawable`，那么只需要用`Canvas`画出来就好了。

接下来观察手势的操作，查看随着手指滑动，控件做出的变化，这里的变化有：

1. 手指按上去时，部分区域变亮（部分区域即为可见区域）
2. 随着手指滑动，相应的数字发生移动，当前角度值也发生改变
3. 离中心越近，透明度越低，且`0°`的下方的点要大一些

好了，我们对这个控件已经分析的很透彻了，根据分析，首先我们要在`View`的`onDraw()`方法中画出构成元素，之后要让它动起来，在`onTouchEvent()`方法中监听手势，改变一些值并重绘我们的`View`，这里很明显，我们要改变的肯定是当前角度这个值。

### 动手

既然有了思路，那么就要马上动手，不然下次忘记掉了怎么办？

#### 画点

首先，先把点给画出来，位置很简单，肯定是从视图中心开始画，往左往右分别画点。

```java
for (int i = 0; i < mPointCount; i++) {
	canvas.getClipBounds(mCanvasClipBounds);
   canvas.drawPoint(mCanvasClipBounds.centerX() + (i - mPointCount / 2) * mPointMargin,
                    mCanvasClipBounds.centerY(), mPointPaint);
}
```

其中`mPointCount`为所要画的点个数，这里默认为51个，`mPointMargin`为两点间的边距，长度为`View`的宽度除以点的个数。

看看效果

![](http://7xrqmj.com1.z0.glb.clouddn.com/degree1.png?imageView2/1/w/360)

嗯，成功的把点给画出来了。

#### 画刻度上方的数字

既然把点给画出来了，根据我们的思路，我们要把数字给画到正确的位置上，画数字当然很简单，这里难的是找到正确的位置。

首先，我们的数字是会移动的，随着当前角度的不同，所展示的数字大小位置都不同。但毫无疑问的是，这个位置（x坐标）肯定是关于当前角度的一个函数。至于具体是一个什么样的函数，相信聪明的你很快就可以分析出来，毕竟只是一个线性关系，这里就直接贴代码了。

```java
private void drawDegreeText(int degrees, Canvas canvas, boolean canReach) {
    canvas.drawText(degrees + "°",
                getWidth() / 2 + mPointMargin * degrees / 2 - mTextWidths[0] / 2 * 3 - mCurrentDegrees / 2 * mPointMargin,
                 getHeight() / 2 - 10,
                 mTextPaint);
    }
}
```

按照我们的效果，我们需要画出-90°~90°的刻度，其中-45°～45°是可到达角度，另外的角度不可到达。

```java
drawDegreeText(0, canvas, true);
drawDegreeText(15, canvas, true);
drawDegreeText(30, canvas, true);
drawDegreeText(45, canvas, true);
drawDegreeText(-15, canvas, true);
drawDegreeText(-30, canvas, true);
drawDegreeText(-45, canvas, true);

drawDegreeText(60, canvas, false);
drawDegreeText(75, canvas, false);
drawDegreeText(90, canvas, false);
drawDegreeText(-60, canvas, false);
drawDegreeText(-75, canvas, false);
drawDegreeText(-90, canvas, false);
```

好了，来看一下效果，可以看到，数字被画在了正确的位置。

![](http://7xrqmj.com1.z0.glb.clouddn.com/degree2.png?imageView2/1/w/360)

#### 画当前角度

这个超级简单呢，位置也特别好确定，上方三角形的Path也非常好知道，但是这里要注意的是，当当前角度为0°左右时，不应该把0°刻度显示出来。

```java
//画当前角度
if (mCurrentDegrees >= 10) {
    canvas.drawText(mCurrentDegrees + "°", getWidth() / 2 - mTextWidths[0], mBaseLine, mTextPaint);
} else if (mCurrentDegrees <= -10) {
    canvas.drawText(mCurrentDegrees + "°", getWidth() / 2 - mTextWidths[0] / 2 * 3, mBaseLine, mTextPaint);
} else if (mCurrentDegrees < 0) {
    canvas.drawText(mCurrentDegrees + "°", getWidth() / 2 - mTextWidths[0], mBaseLine, mTextPaint);
} else {
    canvas.drawText(mCurrentDegrees + "°", getWidth() / 2 - mTextWidths[0] / 2, mBaseLine, mTextPaint);
}

//三角指示器的Path
mIndicatorPath.moveTo(w / 2, h / 2 + mFontMetrics.top / 2 - 18);
mIndicatorPath.rLineTo(-8, -8);
mIndicatorPath.rLineTo(16, 0);
```

还有一个小细节，就是离中心越近，其透明度越来越低，也就是说，我们要根据离中心的位置，来改变画笔`Paint`的`alpha`值，再将点画出。

```java
 for (int i = 0; i < mPointCount; i++) {

     if (i > zeroIndex - 22 && i < zeroIndex + 22 && mScrollStarted) {
         mPointPaint.setAlpha(255);
     } else {
         mPointPaint.setAlpha(100);
     }

     if (i > mPointCount / 2 - 8 && i < mPointCount / 2 + 8
              && i > zeroIndex - 22 && i < zeroIndex + 22) {
         if (mScrollStarted)
              mPointPaint.setAlpha(Math.abs(mPointCount / 2 - i) * 255 / 8);
         else
              mPointPaint.setAlpha(Math.abs(mPointCount / 2 - i) * 100 / 8);
     }
     ……
 }
```

这时，已经很像了，只是还不能动。

![](http://7xrqmj.com1.z0.glb.clouddn.com/degree3.png?imageView2/1/w/360)

#### 动起来

该绘制的部分我们已经都绘制起来了，是时候让这个View动起来了，角度选择器的触摸事件不复杂，我们只需要在我们移动手指的时候根据滑动距离来改变当前角度值并重绘视图就可以看到移动效果了。为什么呢？因为我们之前在画数字的时候，位置都是由当前角度这个值决定的，所以它一发生变化，数字的位置也会发生改变。

```java
@Override
public boolean onTouchEvent(MotionEvent event) {
    switch (event.getAction()) {
     ……
        case MotionEvent.ACTION_MOVE:
            float distance = event.getX() - mLastTouchedPosition;
            ……
            if (distance != 0) {
                onScrollEvent(event, distance);
            }
            break;
        }
      ……
      return true;
}

private void onScrollEvent(MotionEvent event, float distance) {
    mTotalScrollDistance -= distance;
    postInvalidate();
    mLastTouchedPosition = event.getX();
    mCurrentDegrees = (int) ((mTotalScrollDistance * mDragFactor) / mPointMargin);
    if (mScrollingListener != null) {
        mScrollingListener.onScroll(mCurrentDegrees);
    }
}
```

当然你还需要处理一些细节性的东西，比如在数字移动的时候，靠近中心一定范围就会消失（透明度变为0），这些都是很容易控制的，只要改变画笔的透明度就好了，但是正是对细节的把控，才能做出一个效果让用户满意的自定义View。最后，再来看一下效果。

![](http://7xrqmj.com1.z0.glb.clouddn.com/DegreeSeekbar.gif)

#### 扩展

到这里，我们做的角度选择器已经和Google Photos的几乎一模一样了，但是，仅仅这样就够了。不，不够，我们还要继续扩展，为什么只能达到正负45°，我们要所有的范围自由选择，也就是-180°～180°，然后数字颜色啊，点的颜色啊都要让人自由选择。所以我们要扩展我们的角度选择器，把一切可以变化的接口暴露出来。

最后看一下我们多种多样的角度选择器，还是挺好看呀～

![](http://7xrqmj.com1.z0.glb.clouddn.com/DegreeSeekBar2.gif)

### 总结

这次的自定义View相对于前两次的来说，无疑是简单了很多，只运用了最基础的绘图知识和事件机制，但是看起来效果也还不错哦，嘿嘿，反正我特别喜欢这个角度选择器，虽然我还不知道除了裁图页面可以用到还有哪里可以用到。所以说运用最简单的知识，也可以组合出比较复杂的效果，希望大家多发挥自己的想象力，一起自定义出更多好玩的，实用的，酷炫的控件吧。

希望这篇文章对你有帮助，哪怕只是一些启发，我也很开心了。

最后附上源码地址：[https://github.com/wuapnjie/DegreeSeekBar](https://github.com/wuapnjie/DegreeSeekBar)