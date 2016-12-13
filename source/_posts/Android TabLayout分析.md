---
title: Android TabLayout分析
date: 2016-09-08 09:11:58
tags: [Android,Android View]
---

# TabLayout的构造方法分析
在我自己制作小APP时，经常使用TabLayout，于是就想看看它是怎么实现的。
点进TabLayout类查看源码，发现它继承自HorizontalScrollView
```java
public class TabLayout extends HorizontalScrollView
```
的确，当我们将TabLayout的Mode设为MODE_SCROLLABLE时，它的确是横向可滑动的，之前并不知道源码应该怎么看，但感觉应该从构造方法开始看起。接下来看看TabLayout在初始化时做了些什么。
我们都知道，如果在XML中使用TabLayout，会默认使用2个参数的构造函数
```java
public TabLayout(Context context, AttributeSet attrs) {
        this(context, attrs, 0);
}
```
这里调用了3个参数的构造方法，哈哈，好吧，其实大家都这么写，1个参数的构造方法调用2个参数，2个参数的构造方法调用3个的。那么直接看3个参数的构造方法里做了一些
```java
public TabLayout(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);

        // 1 见下方
        ThemeUtils.checkAppCompatTheme(context);
        // 2
        // Disable the Scroll Bar 不显示滚动条
        setHorizontalScrollBarEnabled(false);
        // 3
        // Add the TabStrip
        mTabStrip = new SlidingTabStrip(context);
        super.addView(mTabStrip, 0, new HorizontalScrollView.LayoutParams(
                LayoutParams.WRAP_CONTENT, LayoutParams.MATCH_PARENT));

        //这里省略了很多代码，就是从XML里读取一些自定义的属性
        TypedArray a = context.obtainStyledAttributes(attrs, R.styleable.TabLayout,
                defStyleAttr, R.style.Widget_Design_TabLayout);

       ……

        // 4
        // Now apply the tab mode and gravity
        applyModeAndGravity();
}
```
从构造方法中我们知道了TabLayout在创建时会先检查当前的Activity的Theme是否是Theme.AppCompat系的（上方代码1处），肯定TabLayout依赖了很多在这个主题系里定义的属性，如果不是，这里会抛出一个异常，代码也非常简单
```java
static void checkAppCompatTheme(Context context) {
        TypedArray a = context.obtainStyledAttributes(APPCOMPAT_CHECK_ATTRS);
        final boolean failed = !a.hasValue(0);
        if (a != null) {
            a.recycle();
        }
        if (failed) {
            throw new IllegalArgumentException("You need to use a Theme.AppCompat theme "
                    + "(or descendant) with the design library.");
        }
}
```

然后令横向滚动条消失（代码2处），哈哈，不然多违和。之后创建了一个SlidingTabStrip的实例（代码3处），并把它加进了TabLayout中，那么问题来了，这是个什么东西，于是Ctrl+右键点进去看，发现是一个定义在TabLayout内的继承自LinearLayout的内部类
```java
private class SlidingTabStrip extends LinearLayout
```
哦，是个线性布局，哈哈那肯定是个横向的线性布局，恩，不要问我怎么知道的，聪明的人都知道，看看它的构造方法里做了什么，里面只有一个构造方法。
```java
SlidingTabStrip(Context context) {
            super(context);
            setWillNotDraw(false);
            mSelectedIndicatorPaint = new Paint();
}
```
恩，除了调用父类的构造方法外只做了两件事情，第一件事是告诉夫布局我是一个要自己画自己的线性布局，然后第二件事就是初始化了一支画笔mSelectedIndicatorPaint。从名字中我们可以看出，它要花的是指示器，恩，指示器就是这个东西，下图中的黄条。
![components_tabs_usage_mobile3.png-11.1kB][1]

然后这个内部类里声明了和指示器有关的各种方法，以后会分析的。
回到TabLayout的构造方法，获取了许多自定义的属性后，调用了applyModeAndGravity()这个方法（代码4处），看名字是让我们设置的TabLayout的Mode和Gravity生效，看一下具体做了什么事情。
```java
 private void applyModeAndGravity() {
        int paddingStart = 0;
        if (mMode == MODE_SCROLLABLE) {
            // If we're scrollable, or fixed at start, inset using padding
            paddingStart = Math.max(0, mContentInsetStart - mTabPaddingStart);
        }
        ViewCompat.setPaddingRelative(mTabStrip, paddingStart, 0, 0, 0);

        switch (mMode) {
            case MODE_FIXED:
                mTabStrip.setGravity(Gravity.CENTER_HORIZONTAL);
                break;
            case MODE_SCROLLABLE:
                mTabStrip.setGravity(GravityCompat.START);
                break;
        }

        updateTabViews(true);
}
```
就是如果我们把TabLayout的Mode设为MODE_SCROLLABLE后，且在layout中使用
```xml
app:tabContentStart="100dp"
```
这个属性后，TabLayout会有一个padding_start，效果就是这样
![TabLayout-2.png-3.2kB][2]
在一开始出现了内边距。
PS.在看这个东西源码的时候有ViewCompat这么好的用的类哈。
然后就是设置子View LinearLayout的Gravity。然后调用了updateTabViews(true)这个方法，点进去看看。
```java
    private void updateTabViews(final boolean requestLayout) {
        for (int i = 0; i < mTabStrip.getChildCount(); i++) {
            View child = mTabStrip.getChildAt(i);
            child.setMinimumWidth(getTabMinWidth());
            updateTabViewLayoutParams((LinearLayout.LayoutParams) child.getLayoutParams());
            if (requestLayout) {
                child.requestLayout();
            }
        }
    }
```
恩，很简单，就是把内部自定义的LinearLayout中的子View全部拿出，设置最小的宽度，这个会根据Mode的不同而的到不同的最小宽度,然后设置布局参数重新布局，看看updateTabViewLayoutParams这个方法。
```java
private void updateTabViewLayoutParams(LinearLayout.LayoutParams lp) {
        if (mMode == MODE_FIXED && mTabGravity == GRAVITY_FILL) {
            lp.width = 0;
            lp.weight = 1;
        } else {
            lp.width = LinearLayout.LayoutParams.WRAP_CONTENT;
            lp.weight = 0;
        }
}
```
就是如果Mode为MODE_FIXED或TabLayout的Gravity属性为Fill的话，那么就把LayoutParam的width设为0，weight设为1，这在一个线性布局中就会有几个子View就会几等分，否则就是wrap_content。

好了构造方法分析完了，我们可以知道TabLayout的视图结构是这样子的
![TabLayout-3.svg-7kB][3]
看完构造方法我们还并不知道Item是什么，但是接下来的分析中我们会知道这是什么的。

#TabLayout的基本使用分析
恩，下面是代码
```java
mTabLayout = (TabLayout) findViewById(R.id.tabLayout);
mTabLayout.addTab(mTabLayout.newTab().setText("Item1"));
mTabLayout.addTab(mTabLayout.newTab().setText("Item2"),1);
mTabLayout.addTab(mTabLayout.newTab().setText("Item3"),2,true);
```
效果如下
![TabLayout-4.png-2.6kB][4]
看到我们成功添加了三个Tab，但是有bug，我们在加入Item3时设置了Item3被选中，但是Item1的文字颜色也是选中状态的颜色，有趣。
接下来我们分析TabLayout的addTab的三个方法。首先addTab最重要的参数肯定是Tab，而这个Tab是通过mTabLayout的newTab方法得到的，那么这个Tab是什么的呢，点进去查看，这是一个TabLayout内部的嵌套类。
```java
    /**
     * A tab in this layout. Instances can be created via {@link #newTab()}.
     */
    public static final class Tab {

        /**
         * An invalid position for a tab.
         *
         * @see #getPosition()
         */
        public static final int INVALID_POSITION = -1;

        private Object mTag;
        private Drawable mIcon;
        private CharSequence mText;
        private CharSequence mContentDesc;
        private int mPosition = INVALID_POSITION;
        private View mCustomView;

        private TabLayout mParent;
        private TabView mView;

        private Tab() {
            // Private constructor
        }

        //一些get，set方法
        ……

        private void updateView() {
            if (mView != null) {
                mView.update();
            }
        }

        private void reset() {
            mParent = null;
            mView = null;
            mTag = null;
            mIcon = null;
            mText = null;
            mContentDesc = null;
            mPosition = INVALID_POSITION;
            mCustomView = null;
        }
    }
```        
从代码中可以看出,Tab类是一个类似于Bean的类，储存了各种信息，并提供了仅供内部调用的更新和重置方法，相当与一个信息的载体。且构造方法私有，只能通过TabLayout的newTab()方法获取实例。
从之前的分析我们可以知道，TabLayout肯定把Tab中的所带有的数据添加到了内部自定义的LinearLayout中，而这肯定是在addTab系列方法中完成，点击去查看这些addTab方法，发现调用的是2个参数和3个参数的方法，让我们看看源码。
```java
public void addTab(@NonNull Tab tab, boolean setSelected) {
        if (tab.mParent != this) {
            throw new IllegalArgumentException("Tab belongs to a different TabLayout.");
        }

        addTabView(tab, setSelected);
        configureTab(tab, mTabs.size());
        if (setSelected) {
            tab.select();
        }
}

public void addTab(@NonNull Tab tab, int position, boolean setSelected) {
        if (tab.mParent != this) {
            throw new IllegalArgumentException("Tab belongs to a different TabLayout.");
        }

        addTabView(tab, position, setSelected);
        configureTab(tab, position);
        if (setSelected) {
            tab.select();
        }
}
```
可以看到两个方法是很接近，所以我们只看一个，在addTab方法内部，先进行判断，传入的Tab实例的mParent属性是否为当前TabLayout的实例，说明在TabLayout调用newTab()方法生成Tab实例时初始话了Tab的mParent属性。之后调用了addTabView()和configureTab()这两个方法，让我们来看看这两个方法做了什么。
```java
private void addTabView(Tab tab, int position, boolean setSelected) {
        final TabView tabView = tab.mView;
        mTabStrip.addView(tabView, position, createLayoutParamsForTabs());
        if (setSelected) {
            tabView.setSelected(true);
        }
}
```
oh！在这个私有方法里生成了TabView实例并加入了mTabStrip中，恩，等等，好像漏了什么，tab.mView是什么时候声明的，再回去一看，是在TabLayout的newTab()方法中
```java
 public Tab newTab() {
        Tab tab = sTabPool.acquire();
        if (tab == null) {
            tab = new Tab();
        }
        tab.mParent = this;
        tab.mView = createTabView(tab);
        return tab;
}
```
恩，这个方法就是从TabPool中取出一个Tab，然后在这这里才是真正的构造了一个TabView的实例，并赋给tab.mView。
虽然我们不知道TabView是什么还，但是这肯定就是之前的Item呀，于是TabLayout的视图结构就很清楚了。
![TabLayout-5.svg-7kB][5]

恩，不急，先分析完configureTab()这个方法再去看看TabView是怎么定义的。
```java
private void configureTab(Tab tab, int position) {
        tab.setPosition(position);
        mTabs.add(position, tab);

        final int count = mTabs.size();
        for (int i = position + 1; i < count; i++) {
            mTabs.get(i).setPosition(i);
        }
}
```
恩，就是定义Tab的position属性，并加入到mTabs(ArrayList)这个容器中，并更新所有Tab的position属性，使其正确。
接下来我们看看TabView是如何定义的。
```java
class TabView extends LinearLayout implements OnLongClickListener
```
是一个内部的继承自LinearLayout的内部类。从构造方法中可以看出，这是一个垂直的可点击的线性布局
```java
public TabView(Context context) {
            super(context);
            if (mTabBackgroundResId != 0) {
                setBackgroundDrawable(
                        AppCompatDrawableManager.get().getDrawable(context, mTabBackgroundResId));
            }
            ViewCompat.setPaddingRelative(this, mTabPaddingStart, mTabPaddingTop,
                    mTabPaddingEnd, mTabPaddingBottom);
            setGravity(Gravity.CENTER);
            setOrientation(VERTICAL);
            setClickable(true);
}
```
且，内部有个update方法，会根据Tab中拥有的属性对其内部的View设置Visibility，比如我们上面只设置了Tab的text，所以TabView内部的ImageView和mCustomView就会被设置为GONE,我们只看得到内部的TextView。并且，当我们调用Tab的setXXX()方法都会调用TabView的update()方法来更新视图。

总结一下，TabView中有一个Tab实例，而Tab中也有一个TabView实例，两两对应，Tab作为数据的载体，储存了许多数据，比如要显示的文字，要显示的Icon，或者要显示的自定义View，而TabView负责显示这些数据，TabView内部有许多子View负责显示这些数据。并在某些没有数据时让负责显示这个数据的子View消失不显示。且Tab中的每个数据发生改变时都会提醒TabView，让其做出相应视图上的改变。

# TabLayout与ViewPager的搭配使用分析
恩，我每次都是和ViewPager搭配使用的，从来没有像上面那样用过，使用方法很简单。
```java
 mTabLayout.setupWithViewPager(mPager);
```
传入的为ViewPager的实例。
越简单的使用，说明这个方法内部替我们做了很多事情，肯定是要从这个方法开始分析的。
```java
public void setupWithViewPager(@Nullable final ViewPager viewPager) {
        if (mViewPager != null && mPageChangeListener != null) {
            // If we've already been setup with a ViewPager, remove us from it
            mViewPager.removeOnPageChangeListener(mPageChangeListener);
        }

        if (viewPager != null) {
            final PagerAdapter adapter = viewPager.getAdapter();
            if (adapter == null) {
                throw new IllegalArgumentException("ViewPager does not have a PagerAdapter set");
            }

            mViewPager = viewPager;

            // Add our custom OnPageChangeListener to the ViewPager
            if (mPageChangeListener == null) {
                mPageChangeListener = new TabLayoutOnPageChangeListener(this);
            }
            mPageChangeListener.reset();
            viewPager.addOnPageChangeListener(mPageChangeListener);

            // Now we'll add a tab selected listener to set ViewPager's current item
            setOnTabSelectedListener(new ViewPagerOnTabSelectedListener(viewPager));

            // Now we'll populate ourselves from the pager adapter
            setPagerAdapter(adapter, true);
        } else {
            // We've been given a null ViewPager so we need to clear out the internal state,
            // listeners and observers
            mViewPager = null;
            setOnTabSelectedListener(null);
            setPagerAdapter(null, true);
        }
}
```
看到，如果我们的TabLayout已经和一个ViewPager相互连接，那么先移除PageChangeListener，即解除连接。之后获取传入的ViewPager的Adapter，然后……好吧，最关键的是下面三个方法。
```java
viewPager.addOnPageChangeListener(mPageChangeListener);
// Now we'll add a tab selected listener to set ViewPager's current item
setOnTabSelectedListener(new ViewPagerOnTabSelectedListener(viewPager));
// Now we'll populate ourselves from the pager adapter
setPagerAdapter(adapter, true);
```
给传入的ViewPager设置了PageChangeListener，这个PageChangeListener是在TabLayout中定义的
```java
public static class TabLayoutOnPageChangeListener implements ViewPager.OnPageChangeListener {
        private final WeakReference<TabLayout> mTabLayoutRef;
        private int mPreviousScrollState;
        private int mScrollState;

        public TabLayoutOnPageChangeListener(TabLayout tabLayout) {
            mTabLayoutRef = new WeakReference<>(tabLayout);
        }

        @Override
        public void onPageScrollStateChanged(int state) {
            mPreviousScrollState = mScrollState;
            mScrollState = state;
        }

        @Override
        public void onPageScrolled(int position, float positionOffset,
                int positionOffsetPixels) {
            final TabLayout tabLayout = mTabLayoutRef.get();
            if (tabLayout != null) {
                // Only update the text selection if we're not settling, or we are settling after
                // being dragged
                final boolean updateText = mScrollState != SCROLL_STATE_SETTLING ||
                        mPreviousScrollState == SCROLL_STATE_DRAGGING;
                // Update the indicator if we're not settling after being idle. This is caused
                // from a setCurrentItem() call and will be handled by an animation from
                // onPageSelected() instead.
                final boolean updateIndicator = !(mScrollState == SCROLL_STATE_SETTLING
                        && mPreviousScrollState == SCROLL_STATE_IDLE);
                tabLayout.setScrollPosition(position, positionOffset, updateText, updateIndicator);
            }
        }

        @Override
        public void onPageSelected(int position) {
            final TabLayout tabLayout = mTabLayoutRef.get();
            if (tabLayout != null && tabLayout.getSelectedTabPosition() != position) {
                // Select the tab, only updating the indicator if we're not being dragged/settled
                // (since onPageScrolled will handle that).
                final boolean updateIndicator = mScrollState == SCROLL_STATE_IDLE
                        || (mScrollState == SCROLL_STATE_SETTLING
                        && mPreviousScrollState == SCROLL_STATE_IDLE);
                tabLayout.selectTab(tabLayout.getTabAt(position), updateIndicator);
            }
        }

        private void reset() {
            mPreviousScrollState = mScrollState = SCROLL_STATE_IDLE;
        }
    }
```

恩，就是很简单的，当ViewPager滑动时，让TabLayout滑动到相应的位置。这个Listener使得TabLayout的状态可以更随ViewPager的滑动而做出相应的改变。
接下来看setOnTabSelectedListener(new ViewPagerOnTabSelectedListener(viewPager))这个方法，可以看到关键是传入的ViewPagerOnTabSelectedListener对象，这个也是在TabLayout内部定义的
```java
public static class ViewPagerOnTabSelectedListener implements TabLayout.OnTabSelectedListener {
        private final ViewPager mViewPager;

        public ViewPagerOnTabSelectedListener(ViewPager viewPager) {
            mViewPager = viewPager;
        }

        @Override
        public void onTabSelected(TabLayout.Tab tab) {
            mViewPager.setCurrentItem(tab.getPosition());
        }

        @Override
        public void onTabUnselected(TabLayout.Tab tab) {
            // No-op
        }

        @Override
        public void onTabReselected(TabLayout.Tab tab) {
            // No-op
        }
}
```
可以看到，在onTabSelected()方法中操作了ViewPager的状态，通过这个Listener就使得在TabLayout的状态发生变化时通知ViewPager发生相应的改变。

再来看第三个方法
```java
 private void setPagerAdapter(@Nullable final PagerAdapter adapter, final boolean addObserver) {
        if (mPagerAdapter != null && mPagerAdapterObserver != null) {
            // If we already have a PagerAdapter, unregister our observer
            mPagerAdapter.unregisterDataSetObserver(mPagerAdapterObserver);
        }

        mPagerAdapter = adapter;

        if (addObserver && adapter != null) {
            // Register our observer on the new adapter
            if (mPagerAdapterObserver == null) {
                mPagerAdapterObserver = new PagerAdapterObserver();
            }
            adapter.registerDataSetObserver(mPagerAdapterObserver);
        }

        // Finally make sure we reflect the new adapter
        populateFromPagerAdapter();
}
```
令ViewPager的Adapter注册了数据观察器，即当我们调用adapter的notifyDataSetChanged()之类方法时会使TabLayout也做出数据的变化。最后调用的populateFromPagerAdapter()方法，令改变生效，让TabLayout的Tab数以及Tab的Text与ViewPager相一致。
```java
private void populateFromPagerAdapter() {
        removeAllTabs();

        if (mPagerAdapter != null) {
            final int adapterCount = mPagerAdapter.getCount();
            for (int i = 0; i < adapterCount; i++) {
                addTab(newTab().setText(mPagerAdapter.getPageTitle(i)), false);
            }

            // Make sure we reflect the currently set ViewPager item
            if (mViewPager != null && adapterCount > 0) {
                final int curItem = mViewPager.getCurrentItem();
                if (curItem != getSelectedTabPosition() && curItem < getTabCount()) {
                    selectTab(getTabAt(curItem));
                }
            }
        } else {
            removeAllTabs();
        }
}
```
更新方法也比较简单，就是先清空所有以后的Tab，然后根据ViewPager中的信息创建新的Tab。


  [1]: http://static.zybuluo.com/wuapnjie/folma3ak8fb9hna34s1u8q7h/components_tabs_usage_mobile3.png
  [2]: http://static.zybuluo.com/wuapnjie/009wt6qorxs3epe0jp0u0xv2/TabLayout-2.png
  [3]: http://static.zybuluo.com/wuapnjie/fg80tt1udyqmmxdnsbg9t8vm/TabLayout-3.svg
  [4]: http://static.zybuluo.com/wuapnjie/psv6lmbvd9ahu9sf7ikk641g/TabLayout-4.png
  [5]: http://static.zybuluo.com/wuapnjie/q1j97shk7hshvqeuhpxsve8g/TabLayout-5.svg
