---
title: Android自定义View实战之贴纸StickerView
date: 2016-11-2 19:46:45
tags: [Android]
---

---

本篇文章为利用Matrix自定义View三部曲的第一部曲。

虽然Android内置了许多View供开发者组合和使用，但其多样性还是不足，在很多场景或功能需求下，Android原生自带的控件并不足以实现我们的需求，这时我们就需要自定义满足我们需求的View。

本文会讲解一个自定义View的设计和开发过程，在阅读之前希望大家有最基础的自定义View的知识，以及Matrix类的基本使用。

### 起步

在很多图片社交的应用，例如Lofter、Play、In等应用中，都会有添加各种可爱的贴图到图片上的功能，然后我们可以对图片进行移动、旋转、缩放、翻转之类的操作，本文制作的View正是为了实现这个功能。最终我们将要实现的效果如下图：



![](http://7xrqmj.com1.z0.glb.clouddn.com/stickerview.gif)

项目地址：[https://github.com/wuapnjie/StickerView](https://github.com/wuapnjie/StickerView)

### 简单思考(确定大致思路)

要实现这样的效果，我们肯定需要对图片进行操作，在自定义的View中，我们可以在`onDraw()`方法将我们的图片(通常为Bitmap)画到View上。

```java
 protected void onDraw(Canvas canvas) {
     super.onDraw(canvas);
   	 canvas.drawBitmap(bitmap,matrix,paint);
 }
```

`drawBitmap()`方法有许多重载方法，但是利用Matrix来控制画在View上的图片是最灵活最简单的。（不熟悉Matrix类可以先去了解下，这里就不介绍基础的知识了）

利用Matrix可以方便的控制图片的位置，旋转角度，缩放比。

再看我们的功能，用不同的手势来操作图片，既然利用Matrix可以操作图片，那么我们只需要在View的`onTouchEvent()`方法中监听不同的手势操作，再对其Matrix进行变换，重绘View即可。整个思路流程就很清楚了。

### 仔细思考（决定结构）

有了思路，那么我们就要来考虑我们应该怎么样组织代码，怎么样设计代码的结构。当然这个View并不复杂，设计起来也不复杂。

首先，对于贴纸功能，在没有一张贴纸时就只显示一张图片，而这个功能ImageView已经为我们实现了，于是StickerView应该继承自ImageView，并且重写`onDraw()`和`onTouchEvent()`方法。

其次，因为一张图片上可以添加多张贴纸，而每一张贴纸都需要一个Matrix来控制其相关变换，所以我们可以设计一个类封装一下，方便对贴纸的操作。

```java
public abstract class Sticker {
    protected Matrix mMatrix;
  	public abstract void draw(Canvas canvas);
  	……
}
```

因为贴纸可能是Bitmap，也就是普通的图片，但是我们也可以添加气泡啊，标签啊之类的自定义的Drawable，当然也可能是各种图形，为了其扩展性，这里将Sticker类抽象。扩展的`DrawableSticker`

```java
public class DrawableSticker extends Sticker {
    private Drawable mDrawable;
    private Rect mRealBounds;
    ……
  	@Override
    public void draw(Canvas canvas) {
        canvas.save();
        canvas.concat(mMatrix);
        mDrawable.setBounds(mRealBounds);
        mDrawable.draw(canvas);
        canvas.restore();
    }
  	……
}
```

那么大致的结构就确定了，在View的`onTouchEvent()`中，我们根据手势改变Sticker的Matrix，并在`onDraw()`方法中将Sticker画出。

```java
protected void onDraw(Canvas canvas) {
    super.onDraw(canvas);
  	……
	sticker.draw(canvas);
  	……
}
```

### 实现

在有了思路和一个结构后，大致已经成功了一半，接下来就是一个个功能的实现，和一遍遍的调试了。

由于我们可以添加不止一个Sticker，所以我们的StickerView需要保有对所有添加的Sticker应用，这里可以用一个List集合来储存。而对于当前正在操作的Sticker引用需要额外储存。

因为对于不同的手势，我们所做出的操作不同，那么我们需要在内部声明所有存在的状态和一个当前状态

```java
public class StickerView extends ImageView {
    private enum ActionMode {
        NONE,   //nothing
        DRAG,   //drag the sticker with your finger
        ZOOM_WITH_TWO_FINGER,   //zoom in or zoom out the sticker and rotate the sticker with two finger
        ZOOM_WITH_ICON,    //zoom in or zoom out the sticker and rotate the sticker with icon
        DELETE,  //delete the handling sticker
        FLIP_HORIZONTAL //horizontal flip the sticker
    }
  	private ActionMode mCurrentMode = ActionMode.NONE;

    private List<Sticker> mStickers = new ArrayList<>();
    private Sticker mHandlingSticker;
  	……
}
```

接下来就是一个一个功能实现，但肯定的是，最先需要实现的就是将贴纸添加进来的方法。

#### 添加贴纸

实现起来也很简单，这里就是new一个Sticker对象，并把它加入到我们的List中并重绘，注意，我们默认将Sticker缩放至原来的一半，并放在StickerView中央。

```java
public void addSticker(Drawable stickerDrawable) {
        Sticker drawableSticker = new DrawableSticker(stickerDrawable);

        float offsetX = (getWidth() - drawableSticker.getWidth()) / 2;
        float offsetY = (getHeight() - drawableSticker.getHeight()) / 2;
        drawableSticker.getMatrix().postTranslate(offsetX, offsetY);

        float scaleFactor;
        if (getWidth() < getHeight()) {
            scaleFactor = (float) getWidth() / stickerDrawable.getIntrinsicWidth();
        } else {
            scaleFactor = (float) getHeight() / stickerDrawable.getIntrinsicWidth();
        }
        drawableSticker.getMatrix().postScale(scaleFactor / 2, scaleFactor / 2, getWidth() / 2, getHeight() / 2);

        mHandlingSticker = drawableSticker;
        mStickers.add(drawableSticker);

        invalidate();
}
```

#### 找到贴纸

在我们的贴纸对象被添加进来后我们才可以继续接下来的操作，在我们触摸屏幕时，要判断是否按在贴纸区域，按在哪个贴纸上。实现比较简单，我们的每个Sticker都有一个矩形范围，在经过移动缩放之类的操作后也可以通过Matrix来轻松得到那个矩形区域（`Rect`类），只需要判断这个范围是否包含我们按下的点，而这一步应该在Touch事件的`ACTION_DOWN`事件中进行。

```java
switch (action) {
     case MotionEvent.ACTION_DOWN:
         mCurrentMode = ActionMode.DRAG;
         mDownX = event.getX();
         mDownY = event.getY();
         mHandlingSticker = findHandlingSticker();          
          ……
}
```

其中`findHandlingSticker()`正是做了这样一些事情

```java
private Sticker findHandlingSticker() {
    for (int i = mStickers.size() - 1; i >= 0; i--) {
        if (isInStickerArea(mStickers.get(i), mDownX, mDownY)) {
            return mStickers.get(i);
         }
    }
    return null;
}
```

#### 移动贴纸

找到了我们要操作的Sticker后，我们就可以对其进行操作了，移动操作最为简单，只涉及一根手指，在`ACTION_DOWN`事件中我们记录下当前Sticker的状态和事件起始坐标，在`ACTION_MOVE`事件中，我们利用当前点的坐标计算出实际偏移量，利用Matrix的`postTransition()`方法让Sticker做出随手指的移动。

```java
mMoveMatrix.set(mDownMatrix);
mMoveMatrix.postTranslate(event.getX() - mDownX, event.getY() - mDownY);
mHandlingSticker.getMatrix().set(mMoveMatrix);
```

#### 缩放与旋转贴纸

一般的缩放与旋转操作都是需要两根手指，所以我们需要在`ACTION_POINT_DOWN`事件中监听第二根手指按下。这时我们还需要计算出两根手指之间的距离以及中心点还有角度，因为我们要让Sticker以这个中心点为中心缩放旋转，在`ACTION_MOVE`事件中以新的两指尖距离/起始两指尖距离作为缩放比缩放。以新的角度-起始角度作为旋转角。

```java
switch (action) {
     case MotionEvent.ACTION_POINTER_DOWN:
        mOldDistance = calculateDistance(event);
    	mOldRotation = calculateRotation(event);
        mMidPoint = calculateMidPoint(event);          
          ……
}
```

相应的缩放与旋转，利用Matrix的`postScale`和`postRotate`方法实现

```java
float newDistance = calculateDistance(event);
float newRotation = calculateRotation(event);

mMoveMatrix.set(mDownMatrix);
mMoveMatrix.postScale(newDistance / mOldDistance, newDistance / mOldDistance, mMidPoint.x, mMidPoint.y);
mMoveMatrix.postRotate(newRotation - mOldRotation, mMidPoint.x, mMidPoint.y);

mHandlingSticker.getMatrix().set(mMoveMatrix);
```

#### 添加选中效果

在经过上面的步骤后，我们的StickerView已经可以添加贴纸，用手势操纵贴纸移动，缩放，旋转了，但是我们并没有对选中的贴纸进行特殊处理，因为一般的应用对于选中的贴纸，都会用一个边框围住，并在相应的边框边角显示一些操作按钮。因为这个按钮有图标，所以我们也可以把其作为一个Sticker，只是还需要一个位置的x，y值。

```java
public class BitmapStickerIcon extends DrawableSticker {
    private float x;
    private float y;
  	……
}
```

因为对于每个Sticker的边框及其坐标是很容易获得的，所以我们只需要在`onDraw`方法中在正在处理的Sticker周围画上边框和按钮就可以了。下面的代码获得了选中Sticker的边角坐标，并将操作按钮画在相应位置。

```java
if (mHandlingSticker != null && !mLooked) {

    float[] bitmapPoints = getStickerPoints(mHandlingSticker);

    float x1 = bitmapPoints[0];
    float y1 = bitmapPoints[1];
    float x2 = bitmapPoints[2];
    float y2 = bitmapPoints[3];
    float x3 = bitmapPoints[4];
    float y3 = bitmapPoints[5];
    float x4 = bitmapPoints[6];
    float y4 = bitmapPoints[7];

    canvas.drawLine(x1, y1, x2, y2, mBorderPaint);
    canvas.drawLine(x1, y1, x3, y3, mBorderPaint);
    canvas.drawLine(x2, y2, x4, y4, mBorderPaint);
    canvas.drawLine(x4, y4, x3, y3, mBorderPaint);

    float rotation = calculateRotation(x3, y3, x4, y4);
    //draw delete icon
    canvas.drawCircle(x1, y1, mIconRadius, mBorderPaint);
    mDeleteIcon.setX(x1);
    mDeleteIcon.setY(y1);
    mDeleteIcon.getMatrix().reset();

    mDeleteIcon.getMatrix().postRotate(
                    rotation, mDeleteIcon.getWidth() / 2, mDeleteIcon.getHeight() / 2);
    mDeleteIcon.getMatrix().postTranslate(
                    x1 - mDeleteIcon.getWidth() / 2, y1 - mDeleteIcon.getHeight() / 2);

    mDeleteIcon.draw(canvas);

            //draw zoom icon
    canvas.drawCircle(x4, y4, mIconRadius, mBorderPaint);
    mZoomIcon.setX(x4);
    mZoomIcon.setY(y4);

    mZoomIcon.getMatrix().reset();
    mZoomIcon.getMatrix().postRotate(
                    45f + rotation, mZoomIcon.getWidth() / 2, mZoomIcon.getHeight() / 2);

    mZoomIcon.getMatrix().postTranslate(
                    x4 - mZoomIcon.getWidth() / 2, y4 - mZoomIcon.getHeight() / 2);

    mZoomIcon.draw(canvas);

    //draw flip icon
    canvas.drawCircle(x2, y2, mIconRadius, mBorderPaint);
    mFlipIcon.setX(x2);
    mFlipIcon.setY(y2);

    mFlipIcon.getMatrix().reset();
    mFlipIcon.getMatrix().postRotate(
                    rotation, mDeleteIcon.getWidth() / 2, mDeleteIcon.getHeight() / 2);
    mFlipIcon.getMatrix().postTranslate(
                    x2 - mFlipIcon.getWidth() / 2, y2 - mFlipIcon.getHeight() / 2);

    mFlipIcon.draw(canvas);
}
```

### 总结

这样，我们大致完成了StickerView的所有功能，当然上面并没有太完整的代码，只是一些代码片段，但是已经说明了大致的思路及操作，想了解更多细节可以去查看[源码](https://github.com/wuapnjie/StickerView)。我们在自定义View时，首先最需要的是一个思路，有了思路之后要想其代码结构，在这两块都想好了以后再开发其功能，会事半功倍。

希望可以对你有帮助。如果有什么疑问，可以随时联系我，欢迎提issue和pr。