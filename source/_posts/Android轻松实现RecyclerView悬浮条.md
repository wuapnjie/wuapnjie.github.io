---
title: Android轻松实现RecyclerView悬浮条
date: 2016-11-7 14:46:45
tags: [Android]
---

---

在我们在刷Instagram的动态时，你是否注意到这样一个小小的动效，就是当一条动态（以卡片形式呈现）向上滑动时，动态卡片的头部会始终悬浮在列表最上方，直到下一张动态卡片的头部将它顶掉并替换它悬浮着。言语可能说不清楚，就直接来看一下它的效果好了。

![Instagram的悬浮条](http://7xrqmj.com1.z0.glb.clouddn.com/Instagram.gif)

综合我上面的文字描述加上这张Gif图，我想大家应该知道这是个什么样的效果了吧。那么不废话了，接下来我就来说说一种很简单的实现方法吧。

### 思路

虽然实现起来炒鸡简单，但还是花了我一个多小时的时间思考实现。先说说思考过程吧，那天中午，Instagram给我推了一条消息（哈，就是我最喜欢的偶像金泰妍更新了Ins），于是我就点进去看了，喜欢了之后就开始研究这个效果，我反复地上下滑这个列表，因为Ins的列表有滚动条，我就发现每次滚动条在那个悬浮条附近的时候就会特别短。看到这个现象，敏锐的你是不是察觉到了什么？没错，我感觉这个就像是`FrameLayout`的效果，一个`FrameLayout`里按顺序有列表，悬浮条两个View，悬浮条覆盖在列表的上方，它在合适的时机更新自己的位置，在合适的时机更新自己的信息，然后看上去就形成了一个悬浮的效果。

接下来我们思考的核心就转移到了如何确定并找到这个合适的时机。

再仔细观察上面的Gif图，我们可以确定当第二个列表项的头部距离列表顶端一个悬浮条的距离时，悬浮条随着列表的滑动改变自身的位置，从而看起来像是被顶掉的效果。画一张简单位置示意图

![](http://7xrqmj.com1.z0.glb.clouddn.com/Suspension.png)

那么，数据更新的时机也很容易确定，就是在悬浮条恰好完全被顶掉的时候，更新自己的数据，并移动到列表顶部。

至于如何找到这个时机会在接下来的实现部分讲解。

### 实现

#### 建立布局

如上面所言，就是一个简单的FrameLayout。

```xml
<FrameLayout
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    app:layout_behavior="@string/appbar_scrolling_view_behavior">

    <android.support.v7.widget.RecyclerView
        android:id="@+id/feed_list"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:background="@android:color/white"
        android:scrollbars="vertical" />

    <your-head-layout>
    ……
    </your-head-layout>
</FrameLayout>
```

注意这里`FrameLayout`的第二个child应该为你列表项要悬浮显示的布局。

#### 找到时机

根据我们的思路，我们首先要找到第二个列表项的头部距离列表顶端一个悬浮条的距离时的那个时机，如果我们能找到这个时机，那么第二个时机也相当于找出来了。

这里我们使用的是`RecyclerView`来实现列表，我们都知道`RecyclerView`的列表布局是由`LayoutManager`来确定的，由于一般要实现悬浮条显示效果的列表一般都为线性列表，即我们一般会使用`LinearLayoutManager`。通过`LinearLayoutManager`，我们可以很方便的获取到`RecyclerView`中相应位置的`View`，这里我们需要获取当前悬浮条数据来源的`View`和其下一个数据来源的`View`。这两个`View`有什么用呢？悬浮条显示的信息是来自第一个可见`View`的，而其下方的`View`正是第二个列表项，我们可以获取到它的`top`值。好了接下来就真的很简单了，我们只要给`RecyclerView`加一个`ScrollListener`，并在相应的回调里做之前我们想好的事就ok了，来看一下代码

```java
mFeedList.addOnScrollListener(new RecyclerView.OnScrollListener() {
    @Override
    public void onScrollStateChanged(RecyclerView recyclerView, int newState) {
        super.onScrollStateChanged(recyclerView, newState);
        mSuspensionHeight = mSuspensionBar.getHeight();
    }

    @Override
    public void onScrolled(RecyclerView recyclerView, int dx, int dy) {
        super.onScrolled(recyclerView, dx, dy);
        //找到列表第二个可见的View
        View view = linearLayoutManager.findViewByPosition(mCurrentPosition + 1);
        if (view == null) return;
        //判断View的top值是否小于悬浮条的高度
        if (view.getTop() <= mSuspensionHeight) {
          	//被顶掉的效果
            mSuspensionBar.setY(-(mSuspensionHeight - view.getTop()));
        } else {
            mSuspensionBar.setY(0);
        }
		
      	//判断是否需要更新悬浮条
        if (mCurrentPosition != linearLayoutManager.findFirstVisibleItemPosition()) {
            mCurrentPosition = linearLayoutManager.findFirstVisibleItemPosition();
            mSuspensionBar.setY(0);
			//更新悬浮条
            updateSuspensionBar();
        }
    }
});
```

**Tips**：其中`mCurrentPosition`为悬浮条信息来自的那个列表项在`RecyclerView`的位置。还有这里的`ScrollListener`可以添加多个，在`RecyclerView`中会检查所有的`ScrollListener`并触发。

#### One more thing...

接下来，我们还需要……开玩笑，哪来的One more thing，我们已经完成了？什么？这么快？这么一点代码？恩，没错，就是只要这么一点代码就好了，我们来看一下最后我们实现的效果（当然最终效果的好坏还是取决与你列表项的布局，比如在Ins里这个效果就很好看呢～）

![](http://7xrqmj.com1.z0.glb.clouddn.com/SuspensionBar.gif)

### 结语

哈哈，是不是很简单呢，最后再说一下封装的事，本来我是想封装一下的，由于每个人的列表布局都不一样，数据更新方式也不一样，就不封装了，是的，我水平不行，虽然我不想承认～不过代码真心特别少哦，源码地址：

希望这篇文章可以对你有帮助，我也会继续努力的。
