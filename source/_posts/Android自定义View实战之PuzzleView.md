---
title: Android自定义View实战之拼图PuzzleView
date: 2016-11-2 19:47:45
tags: [Android]
---

---

本篇文章为利用Matrix自定义View的第二篇，第一篇见[Android自定义View实战之StickerView](http://www.jianshu.com/p/fc37baa97fc3)

在阅读本篇文章之前，希望大家有基本的自定义View知识和Matrix的知识，当然最好阅读了前一篇，因为很多东西是相通的，本文的重点在于前期的思考，至于具体实现细节可以不看，选择看[源码](https://github.com/wuapnjie/PuzzleView)。

### 起步

在图片的处理软件中，拼图是很常见的一种处理方法，我最喜欢Layout for Instagram的拼图效果，简单却又足够强大，拼图方式多种多样可以对图片进行水平垂直翻转，移位，移动，缩放，改变大小之类的操作，看到这样的操作。本文制作的View正是为了实现这个功能。先看最终我们实现的效果。

**多种布局**

![](http://7xrqmj.com1.z0.glb.clouddn.com/puzzle2.mp4_1472025098.gif?imageView2/2/w/200)

**具体布局编辑**

![](http://7xrqmj.com1.z0.glb.clouddn.com/gif-demo1.gif?imageView2/2/w/200)

项目地址：[https://github.com/wuapnjie/PuzzleView](https://github.com/wuapnjie/PuzzleView)

### 确定思路

在前面介绍中，我们知道这一次我们还是对图片的一系列变换操作，那么这次我们的实现思路也是在`onTouchEvent()`中根据手势控制对应的`Matrix`来对所画在View上的图片进行操作。

再仔细看我们的效果，在一个View中我们可能要画上许多张图片，但是位置都不同，且互相不会覆盖，那么可以看出我们对View进行了分割，分成不同的矩形，了解`canvas`的同学知道，`canvas`可以先进行一系列变换后再进行绘制，绘制完成后恢复，这次利用的就是`canvas`的`clipRect()`方法将`canvas`分成不同的矩形区域进行绘制，先来看看大致效果可不可以达到我们的预期。

```java
@Override
protected void onDraw(Canvas canvas) {
    super.onDraw(canvas);

    canvas.save();
    canvas.clipRect(0, 0, getWidth() / 2, getHeight());
    canvas.drawBitmap(mBitmapOne, 0, 0, mBitmapPaint);
    canvas.restore();

    canvas.save();
    canvas.clipRect(getWidth() / 2, 0, getWidth(), getHeight());
    canvas.drawBitmap(mBitmapTwo, 0, 0, mBitmapPaint);
    canvas.restore();
}
```

![](http://7xrqmj.com1.z0.glb.clouddn.com/PoiDemo-Rect.png?imageView2/2/w/360)

可以看到，这样是可以达到我们想要的图片排列方式的，只需要对图片进行矩阵操作，让其适应给定的矩形区域就好了。

那么第一步的思路超不多就想好了，我们做到了如何在一个View中排列多张图片，接下来要思考如何分割外围的矩形（View的边界矩形）。

我们知道Android内置了`Rect`类，用上下左右四个坐标确定一个矩形，一个大的矩形可以很容易的分为许多小的矩形，类似这样

 ![rect](http://7xrqmj.com1.z0.glb.clouddn.com/rect.png)

一个大的矩形被分为三个小矩形。但是这个内置的`Rect`类真的能帮助我们完成效果吗？

答案是不能的，虽然内置的`Rect`类可以成功帮助我们确定每张图片的位置，令图片被画在正确的位置上，但是有一点致命的是，它内部是由上下左右四个坐标确定的，仔细看我们要实现的效果，在随着我们手指对矩形边线的移动，大矩形内的小矩形大小边界是在改变的，而且收到影响的矩形肯定大于等于2个，那么我们要改变坐标的矩形也就会大于等于2个，编码上会复杂且容易出错，所以我们不能单单只用`Rect`类来确定边界。我们必须在抽象出一种新的模型来确定图片的矩形区域并方便数据更新变化。

在反复把玩Layout for Instagram后（因为当时我还没做出这个View，一直拿Layout研究，希望你也可以去多玩一下），并把它的所有布局都在纸上画了一遍，我发现了很关键的一点，也是这个自定义View最关键的一部。它的线很重要（当我们点击其中一张图片后，它会成为选中状态，那个线是高亮的，引人注意哦），我们每次移动的时那一根线，而一个矩形可以被一根直线或横线划分成两个矩形，而四根线可以确定一个矩形范围，两个矩形可以共享一根线，线的位置改变，共享这根线的所有矩形的大小范围都会改变。类似这样

![](http://7xrqmj.com1.z0.glb.clouddn.com/rect2.png)

* line1,line2,line4,line5组成了Rect1
* line2,line3,line4,line5组成了Rect2
* Rect1和Rect2共享line2，line4，line5
* 移动了line2后，Rect1和Rect2均收到影响

希望大家理解这幅图，这是本次自定义View的关键。

那么整理一下大致思路，我们要用线将View的边界分成许多个小矩形，并让图片画在这些小矩形上，之后同上一篇文章一致，根据我们的手势控制对应图片的`Matrix`来控制图片的相应动作。

### 建立模型

既然思路已经确定了，那么我们就要来确定我们的代码结构和相应的模型类。上面讲我们要用线来分割矩形，而Android原生是没有Line这个模型类的，于是我们要自己抽象一个。那么线是怎么组成的呢？很简单，在坐标系中，两点确定一根直线，所以我们要有两个点`PointF`，因为我们只用横线或直线，所以只抽象了两个方向，斜线不考虑（本效果只需要直线和横线）。

```java
public class Line {

    public enum Direction {
        HORIZONTAL,
        VERTICAL
    }

    /**
     * for horizontal line, start means left, end means right
     * for vertical line, start means top, end means bottom
     */
    final PointF start;
    final PointF end;

    private Direction direction = Direction.HORIZONTAL;
  	……
}
```

但是这么几个属性真的够用吗？在我试验了之后发现是不够的，我们还需要另外四个属性，是四根其他的线，两根确定其移动范围的线，两根顶点依附的线。

![](http://7xrqmj.com1.z0.glb.clouddn.com/rect3.png)

于是我们Line的模型类就可以去确定了。

```java
public class Line {

    public enum Direction {
        HORIZONTAL,
        VERTICAL
    }

    /**
     * for horizontal line, start means left, end means right
     * for vertical line, start means top, end means bottom
     */
    final PointF start;
    final PointF end;

    private Direction direction = Direction.HORIZONTAL;

    private Line attachLineStart;
    private Line attachLineEnd;

    private Line mUpperLine;
    private Line mLowerLine;
	……
}
```

那么我们就可以确定一个边界`Border`类，它由4条`Line`构成，并可方便的导出`Rect`对象方便我们摆放图片。

```java
class Border {
    Line lineLeft;
    Line lineTop;
    Line lineRight;
    Line lineBottom;
	……
}
```

接下来就要思考如何支持多样化布局，当然要提供接口供使用者自定义，所以我们要抽象出一个拼图布局类`PuzzleLayout`，这个类要有个抽象方法支持我们自定义布局，并提供一些简单的方法帮助我们快速布局，并且应该保有所有的边界`Border`和`Line`对象，方便进行管理和更新信息。

```java
public abstract class PuzzleLayout {
	……
    private Border mOuterBorder;

    private List<Border> mBorders = new ArrayList<>();
    private List<Line> mLines = new ArrayList<>();
    private List<Line> mOuterLines = new ArrayList<>(4);
  	……
    public abstract void layout();
  	……
}
```

至于图片对象，同上一篇文章一样，每张图片需要一个`Matrix`对象进行控制，只是在这之上还要保有一个边界`Border`的引用。这里就不贴了。

这样，我们所有的模型就已经确定了。大致关系就是，每个`PuzzleView`的布局方式由`PuzzleLayout`决定，`PuzzleLayout`可自定义布局，由一系列的边界`Border`组成，而`Border`则由一系列的`Line`组成。

### 具体实现

由于许多东西的关键都是思路和建模，大家理解了这个思路并建立了正确方便的模型后，实现起来就异常容易了，只是在预定的轨道上开车到终点就好了，其实后面的内容已经不重要了。

#### 布局方式的确定

起初，我们要先把布局方式确定才可以决定画多少张图片上去，所以布局方式是最先要被解决的功能。

大家都知道，一根直线可以把一个矩形分成左右两个矩形，一根横线可以把一个矩形分成上下两个矩形，所以我们可以提供一个`addLine()`方法提供分割布局，将增加的`Line`和`Border`添加至集合。

```java
protected List<Border> addLine(Border border, Line.Direction direction, float ratio) {
    mBorders.remove(border);
    Line line = BorderUtil.createLine(border, direction, ratio);
    mLines.add(line);

    List<Border> borders = BorderUtil.cutBorder(border, line);
    mBorders.addAll(borders);

    updateLineLimit();
    Collections.sort(mBorders, mBorderComparator);

    return borders;
}
```

当然只有这么一个方法布局还是不怎么方便的哈，所以我还添加了许多方法方便布局，比如一个十字可以把一个矩形分割成四个矩形，一个螺旋可以把一个矩形分割成五个矩形。提供的方法大致就如下图所示

![](http://7xrqmj.com1.z0.glb.clouddn.com/puzzle.png)

举个例子:

```java
@Override
public void layout() {
    addLine(getOuterBorder(), Line.Direction.VERTICAL, 1f / 2);
    cutBorderEqualPart(getBorder(1), 4, Line.Direction.HORIZONTAL);
    cutBorderEqualPart(getBorder(0), 3, Line.Direction.HORIZONTAL);
}
```

之后我们看一下这种布局分割的效果

![](http://7xrqmj.com1.z0.glb.clouddn.com/puzzle-layout-demo.png?imageView2/2/w/360)

#### 图片位置的确立与放置

到这里，我们已经可以自定义各种各样的布局了，一个View已经被我们分割成了许多小的矩形区域，接下来我们就要把图片给画上去，但不是随便画，我们需要让图片在对应的矩形以`centerCrop`的方式显示，不然我们看到的就不是图片的重要区域。那么怎么样才可以做到呢？由于每个矩形的位置我们都是知道的，所以我们只需要将图片的中心移动到对应矩阵的中心，按`centerCrop`的缩放规则让图片中心缩放就好了。这些就是`Matrix`的基本应用了，这里就不重复说明了，至于`centerCrop`的缩放比也很好计算，不会的话，看一下`ImageView`的源码就好了。

下面的代码是生成让图片已对应`Border`正确显示的`Matrix`生成 

```java
static Matrix createMatrix(Border border, int width, int height, float extraSize) {
        final RectF rectF = border.getRect();

        Matrix matrix = new Matrix();

        float offsetX = rectF.centerX() - width / 2;
        float offsetY = rectF.centerY() - height / 2;

        matrix.postTranslate(offsetX, offsetY);

        float scale;

        if (width * rectF.height() > rectF.width() * height) {
            scale = (rectF.height() + extraSize) / height;
        } else {
            scale = (rectF.width() + extraSize) / width;
        }

        matrix.postScale(scale, scale, rectF.centerX(), rectF.centerY());

    return matrix;
}
```

将图片画上去后的效果，是不是效果很好呀？

![](http://7xrqmj.com1.z0.glb.clouddn.com/20160824_164657.jpg?imageView2/2/w/360)

#### 图片移动旋转缩放翻转

这个功能和上一篇所讲的方法一致，在`onTouchEvent()`中监听不同的手势，对对应图片的`Matrix`做出相关操作即可，这里就不重复说明了，比较基础。

#### 线的移动

看效果图，这个布局并不是不变的，我们可以通过对可移动线的移动，可以使一些边界变大，另一些边界变小，同时令图片适应边界的变化。这时候模型的正确建立就大大地简化了我们的编码效率。

首先，我们找到我们是否触摸在线上，因为内部的线对象必然会被2个以上的边界引用，当这条线的信息改变时，对应的边界也会马上得知，并改变其边界区域，这样我们就可以很方便的重新画出边界，我们就只要更新受影响区域图片的`Matrix`即可。

```java
moveLine(event); //移动线
mPuzzleLayout.update(); //更新PuzzleLayout内Border信息
updatePieceInBorder(event); //更新图片Matrix信息以适应变化
```

#### 图片位置交换

图片之间的相对位置是可以改变的，按照正常的逻辑也是当我们长按一张图片时，那张图片会悬浮，然后移动到要交换位置的图片，释放手指就交换成功了。那么问题就是这个悬浮起来的效果，这里用全图显示加个半透明来表示，利用`Canvas`的相关方法实现及其容易。

```java
if (mHandlingPiece != null && mCurrentMode == Mode.SWAP) {
    mHandlingPiece.draw(canvas, mBitmapPaint, 128);
    if (mReplacePiece != null) {
        drawSelectedBorder(canvas, mReplacePiece);
    }
}
```

#### 图片翻转

这个同样利用`Matrix`可以轻松实现，不赘述。

```java
matrix.postScale(-1, 1, px, py); //水平翻转
matrix.postScale(1, -1, px, py); //垂直翻转
```

#### 尾声

到这里，我们所要实现的功能已经基本全部实现，剩下的就是完善细节，应该提供怎么样的接口供外部操作，只需要慢慢调试即可，感兴趣的同学可以去看一下[源码](https://github.com/wuapnjie/PuzzleView)。

### 总结

这次自定义的View相对于上一次的[StickerView](https://github.com/wuapnjie/StickerView)来说，无疑是复杂了很多，我们需要建立更复杂的模型，但是所运用的核心类是一直的，`Canvas`和`Matrix`类，同上一篇一样，我还是要强调思考与建模的重要性，万事开头难，前期的思考无疑是最难的，也占据了整个项目大部分的时间(我花了两周思考，呜呜，可能我太笨了)。

希望阅读完这篇文章后，可以对你有一些帮助，有什么问题或不懂可以随时联系我，欢迎骚扰。

最近闲下来了，写点文章记录之前的学习并巩固我的基础知识，希望同大家一起进步！