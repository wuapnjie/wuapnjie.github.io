---
title: Daily Android
date: 2016-09-08 09:03:26
tags: [Android,Daily]
---

这篇文章会记录我日常在Android的一些小技巧。方便以后进行回顾。这篇文章以后应该会越来越长吧。

1.关于CardView在5.0以上的使用，由于5.0以下系统和5.0系统的CardView的内容区域大小不同，所以在5.0以上要添加
```xml
app:cardUseCompatPadding="true"
```
这样一个属性，例如
```xml
<android.support.v7.widget.CardView
    android:id="@+id/media_card_view"
    android:layout_width="match_parent"
    android:layout_height="130dp"
    app:cardBackgroundColor="@android:color/white"
    app:cardElevation="2dp"
    app:cardUseCompatPadding="true"
    >
...
</android.support.v7.widget.CardView>
```

2.关于在fragment中的网络加载问题,在fragment的onAttach()方法之前会调用setUserVisibleHint(isVisibleToUser: Boolean)方法，此时isVisibleToUser为false，而当fragment可见时又会调用，此时isVisibleToUser为true，所以可以在此方法中调用相关的网络加载
```java
@Override
public void setUserVisibleHint(boolean isVisibleToUser) {
    super.setUserVisibleHint(isVisibleToUser);
    if (isVisibleToUser) {
        load();
        //相当于Fragment的onResume
    } else {
        cancel();
        //相当于Fragment的onPause
    }
}
```

3.关于Picasso的磁盘缓存功能，Picasso的磁盘缓存功能是有HttpClient的磁盘缓存做到的，它会先搜索OKHttpClient是否存在
```java
 static Downloader createDefaultDownloader(Context context) {
    try {
      Class.forName("com.squareup.okhttp.OkHttpClient");
      return OkHttpLoaderCreator.create(context);
    } catch (ClassNotFoundException ignored) {
    }
    return new UrlConnectionDownloader(context);
  }
```
但是现在OKHttp3.0+后的包名不符合，因为Picasso只有2.5.2，所以使用时可以用OKHttp2.5+的版本

4.改变选中状态和未选中状态的TabLayout的背景颜色
定义一个Selector的drawable
enshrine_tab_color_selector.xml
```xml
<?xml version="1.0" encoding="utf-8"?>
<selector xmlns:android="http://schemas.android.com/apk/res/android">
    <item android:drawable="@color/enshrine_tab_selected" android:state_selected="true"/>
    <item android:drawable="@color/enshrine_tan_unselected"/>
</selector>
```
之后设置TabLayout的自定义属性即可
```xml
app:tabBackground="@drawable/enshrine_tab_color_selector"
```

5.dpToPixel
```java
public static int dpToPixel(Context context, int dp) {
    DisplayMetrics displayMetrics = context.getResources().getDisplayMetrics();
    return dp < 0 ? dp : Math.round(dp * displayMetrics.density);
}
```

pixelToDp
```java
public static int pixelToDp(Context context, int pixel) {
    DisplayMetrics displayMetrics = context.getResources().getDisplayMetrics();
    return pixel < 0 ? pixel : Math.round(pixel / displayMetrics.density);
}
```

6.使用枚举类型来表示常量
比如在制作play时，有10个表情，每个表情对应不同的资源文件，以前我都是用常量来表示
```java
public static final int EMOJI_CRY = R.drawable.ic_cry;
……
```
但是这样写需要很多代码，而且在进行判断时也要编写许多诸如switch，if elseif之类的代码，大大降低效率。于是我们可以通过枚举类型来表示这样的常量。
```java
public enum EmojiType {
    None(""),
    Laugh("laugh"),
    Shock("shock"),
    Cry("cry"),
    Look("look"),
    Love("love"),
    Mei("mei"),
    Chu("chu"),
    Wu("wu"),
    Meng("meng"),
    Heart("heart");

    private String code;

    private EmojiType(String code) {
        this.code = code;
    }


    public static EmojiType buildEmoji(String str) {
        if (str != null) {
            for (EmojiType b : EmojiType.values()) {
                if (str.equalsIgnoreCase(b.code)) {
                    return b;
                }
            }
        }
        return EmojiType.None;
    }

    public int emojiImage() {
        switch (this) {
            case None:  return 0;
            case Laugh: return R.drawable.ic_emoji_laugh;
            case Shock: return R.drawable.ic_emoji_amazing;
            case Cry: return R.drawable.ic_emoji_kuxiao;
            case Look: return R.drawable.ic_emoji_none;
            case Love: return R.drawable.ic_emoji_love;
            case Mei: return R.drawable.ic_emoji_beauty;
            case Chu: return R.drawable.ic_emoji_chu;
            case Wu: return R.drawable.ic_emoji_sex;
            case Meng: return R.drawable.ic_emoji_meng;
            case Heart: return R.drawable.ic_emoji_heart;
            default: return 0;
        }
    }

}
```
比如这样，我们只需根据不同的字符串来构造不同的EmojiType，因为它提供了工厂构造方法，当我们需要使用表情图片时，也只需要调用
```java
EmojiType.emojiImage();
```
即可，避免写了许多重复冗余的代码。

7.关于Dialog的显示动画
这次在制作play时，有一个从上出现，从下消失的用户卡片，这里我想到用Dialog来进行实现，于是了解了一下Dialog的自定义View，选择主题以及出场动画和消失动画。
下面定义了一个Dialog的style，父style为Theme.AppCompat.DayNight.Dialog。并覆盖了之前的默认动画style，由我们自己编写。
```xml
<style name="UserInfoDialog" parent="Theme.AppCompat.DayNight.Dialog">
    <item name="android:windowAnimationStyle">@style/UserInfoDialogAnimate</item>
</style>

<style name="UserInfoDialogAnimate">
    <item name="android:windowEnterAnimation">@anim/dialog_user_info_show</item>
    <item name="android:windowExitAnimation">@anim/dialog_user_info_dismiss</item>
</style>
```

出场动画
```xml
<?xml version="1.0" encoding="utf-8"?>
<set xmlns:android="http://schemas.android.com/apk/res/android"
    android:duration="300">
    <translate
        android:fromYDelta="-100%"
        android:toYDelta="0%" />

    <alpha
        android:fromAlpha="0"
        android:toAlpha="1" />
</set>
```

退场动画
```xml
<?xml version="1.0" encoding="utf-8"?>
<set xmlns:android="http://schemas.android.com/apk/res/android"
    android:duration="300">
    <translate
        android:fromYDelta="0%"
        android:toYDelta="100%" />

    <alpha
        android:fromAlpha="1"
        android:toAlpha="0" />
</set>
```
接下来要在自定义的Dialog中进行设置
```java
public class UserInfoDialog extends Dialog {
    ……
    public UserInfoDialog(Context context) {
        this(context, R.style.UserInfoDialog);
    }

    public UserInfoDialog(Context context, int themeResId) {
        super(context, themeResId);
    }

    protected UserInfoDialog(Context context, boolean cancelable, OnCancelListener cancelListener) {
        super(context, cancelable, cancelListener);
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.dialog_user_info);

        initView();
    }

    private void initView() {
        ……
    }
}
```
这样一个带动画的自定义Dialog就制作完成了。

8.关于PopupWindow的只用以及注意点
PopupWindow的使用方法与Dialog的使用方法想似
```java
public class PraisePopupWindow {
    private WeakReference<Context> mContextWeakReference;
    private int mOffsetY;
    ……

    public PraisePopupWindow(Context context) {
        mContextWeakReference = new WeakReference<>(context);
        final View view = LayoutInflater.from(context).inflate(R.layout.popup_window_praise, null);
        mPopupWindow = new PopupWindow(view, ViewGroup.LayoutParams.WRAP_CONTENT, ViewGroup.LayoutParams.WRAP_CONTENT);
        mPopupWindow.setOutsideTouchable(true);
        mPopupWindow.setBackgroundDrawable(new ColorDrawable(Color.TRANSPARENT));
        mOffsetY = DipPixelUtil.dip2px(context, 72f);

        initView();
    }


    private void initView() {
        ……
    }

    public void showOrDismiss(View view) {
        if (mPopupWindow.isShowing()) {
            mPopupWindow.dismiss();
        } else {
            mPopupWindow.showAsDropDown(view, 0, -(view.getHeight() + mOffsetY));
        }
    }

    public void dismiss() {
        if (isShowing()) {
            mPopupWindow.dismiss();
        }
    }

    public boolean isShowing() {
        return mPopupWindow.isShowing();
    }

}
```
不同与自定义Dialog，PopupWindow在构造方法中就添加进了自定义的View，并设置了布局属性。
其中需要注意的是这两行代码
```java
mPopupWindow.setOutsideTouchable(true);
//or mPopupWindow.setBackgroundDrawable(new BitmapDrawable());
mPopupWindow.setBackgroundDrawable(new ColorDrawable(Color.TRANSPARENT));
```
第一行声明PopupWindow接收外部的触摸事件，但是并不会接收到外部的触摸事件后就dismiss。只有当我们给其设置一个BackgroundDrawable时，外部的触摸事件才会令其消失。

9.Android WebView相关配置，使其支持视频播放
```java
WebSettings ws= webView.getSettings();
        ws.setJavaScriptEnabled(true);
        ws.setAllowFileAccess(true);
        ws.setDatabaseEnabled(true);
        ws.setDomStorageEnabled(true);
        ws.setSaveFormData(false);
        ws.setAppCacheEnabled(true);
        ws.setCacheMode(WebSettings.LOAD_DEFAULT);
        ws.setLoadWithOverviewMode(false); //<==== 一定要设置为false，不然有声音没图像
        ws.setUseWideViewPort(true);

```
