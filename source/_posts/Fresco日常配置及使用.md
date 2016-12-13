---
title: Fresco日常配置及使用
date: 2016-09-08 09:14:56
tags: [Android,Fresco]
---

Fresco是Facebook出品的一个超级强大的开源图片加载库，支持Gif，Webp格式的图片加载，支持渐进式显示，支持……总之支持很多东西,2333。

###Fresco配置
Fresco的配置很重要，配置的不好就很容易造成性能上的问题

在Application.java中配置
这里是我的配置，使用了OkHttp3作为网络请求库

**gradle配置**
```gradle
compile 'com.facebook.fresco:fresco:0.11.0'
compile 'com.facebook.fresco:imagepipeline-okhttp3:0.11.0'
```
**java代码配置**
```java
        //这里是添加Fresco的日志
Set<RequestListener> requestListeners = new HashSet<>();
requestListeners.add(new RequestLoggingListener());

//当内存紧张时采取的措施
MemoryTrimmableRegistry memoryTrimmableRegistry = NoOpMemoryTrimmableRegistry.getInstance();
memoryTrimmableRegistry.registerMemoryTrimmable(trimType -> {
final double suggestedTrimRatio = trimType.getSuggestedTrimRatio();

Log.e("Fresco", String.format("onCreate suggestedTrimRatio : %d", suggestedTrimRatio));
if (MemoryTrimType.OnCloseToDalvikHeapLimit.getSuggestedTrimRatio() == suggestedTrimRatio
        || MemoryTrimType.OnSystemLowMemoryWhileAppInBackground.getSuggestedTrimRatio() == suggestedTrimRatio
        || MemoryTrimType.OnSystemLowMemoryWhileAppInForeground.getSuggestedTrimRatio() == suggestedTrimRatio
        ) {
    //清除内存缓存
    Fresco.getImagePipeline().clearMemoryCaches();
//                Fresco.getImagePipeline().clearCaches();
}
});

        //小图片的磁盘配置,用来储存用户头像之类的小图
DiskCacheConfig diskSmallCacheConfig = DiskCacheConfig.newBuilder(this)
        .setBaseDirectoryPath(this.getCacheDir())//缓存图片基路径
        .setBaseDirectoryName(getString(R.string.app_name))//文件夹名
        .setMaxCacheSize(20 * ByteConstants.MB)//默认缓存的最大大小。
        .setMaxCacheSizeOnLowDiskSpace(10 * ByteConstants.MB)//缓存的最大大小,使用设备时低磁盘空间。
        .setMaxCacheSizeOnVeryLowDiskSpace(5 * ByteConstants.MB)//缓存的最大大小,当设备极低磁盘空间
        .build();

ImagePipelineConfig imagePipelineConfig = OkHttpImagePipelineConfigFactory
        .newBuilder(this, client)
        .setDownsampleEnabled(true)
        .setResizeAndRotateEnabledForNetwork(true)
        .setRequestListeners(requestListeners)
        .setBitmapMemoryCacheParamsSupplier(new LolipopBitmapMemoryCacheSupplier((ActivityManager) getSystemService(ACTIVITY_SERVICE)))
        .setMemoryTrimmableRegistry(memoryTrimmableRegistry)
        .setSmallImageDiskCacheConfig(diskSmallCacheConfig)
        .setBitmapsConfig(Bitmap.Config.RGB_565)
        .build();

Fresco.initialize(this, imagePipelineConfig);
```
因为在Android5.0以上的机子不能肆意使用匿名内存区域，所以要特别区分一下
```java
import android.app.ActivityManager;
import android.os.Build;

import com.facebook.common.internal.Supplier;
import com.facebook.common.util.ByteConstants;
import com.facebook.imagepipeline.cache.MemoryCacheParams;

/**
 * Created by snowbean on 16-7-2.
 */
public class LolipopBitmapMemoryCacheSupplier implements Supplier<MemoryCacheParams> {
    private ActivityManager activityManager;

    public LolipopBitmapMemoryCacheSupplier(ActivityManager activityManager) {
        this.activityManager = activityManager;
    }

    @Override
    public MemoryCacheParams get() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
            return new MemoryCacheParams(getMaxCacheSize(), 56, Integer.MAX_VALUE,
                    Integer.MAX_VALUE,
                    Integer.MAX_VALUE);
        } else {
            return new MemoryCacheParams(
                    getMaxCacheSize(),
                    256,
                    Integer.MAX_VALUE,
                    Integer.MAX_VALUE,
                    Integer.MAX_VALUE);
        }
    }

    private int getMaxCacheSize() {
        final int maxMemory = Math.min(activityManager.getMemoryClass() * ByteConstants.MB, Integer.MAX_VALUE);

        if (maxMemory < 32 * ByteConstants.MB) {
            return 4 * ByteConstants.MB;
        } else if (maxMemory < 64 * ByteConstants.MB) {
            return 6 * ByteConstants.MB;
        } else {
            if (Build.VERSION.SDK_INT <= Build.VERSION_CODES.GINGERBREAD) {
                return 8 * ByteConstants.MB;
            } else {
                return maxMemory / 4;
            }
        }
    }
}
```

###Fresco图片加载
为了使用方便，将图片加载都使用静态方法作为工具类写在一起
```java
import android.content.Context;
import android.net.Uri;
import android.text.TextUtils;
import android.util.Log;

import com.facebook.binaryresource.BinaryResource;
import com.facebook.binaryresource.FileBinaryResource;
import com.facebook.cache.common.CacheKey;
import com.facebook.drawee.backends.pipeline.Fresco;
import com.facebook.drawee.backends.pipeline.PipelineDraweeController;
import com.facebook.drawee.view.SimpleDraweeView;
import com.facebook.imagepipeline.common.Priority;
import com.facebook.imagepipeline.common.ResizeOptions;
import com.facebook.imagepipeline.core.ImagePipelineFactory;
import com.facebook.imagepipeline.request.ImageRequest;
import com.facebook.imagepipeline.request.ImageRequestBuilder;

import java.io.File;

/**
 * Created by snowbean on 16-7-2.
 */
public class PhotoUtil {
    private static final String TAG = PhotoUtil.class.getSimpleName();

    private PhotoUtil() {

    }

    //Fresco
    public static void display(SimpleDraweeView draweeView, String url) {
        if (TextUtils.isEmpty(url)) {
            Log.e(TAG, "display: error the url is empty");
            return;
        }
        draweeView.setImageURI(url);
    }

    public static void display(SimpleDraweeView draweeView, File file) {
        if (file == null) {
            Log.e(TAG, "display: error the file is empty");
            return;
        }
        Uri uri = Uri.fromFile(file);
        if (uri == null) return;
        draweeView.setImageURI(uri);
    }

    public static void display(SimpleDraweeView draweeView, Uri uri) {
        if (uri == null) {
            Log.e(TAG, "display: error the url is empty");
            return;
        }
        draweeView.setImageURI(uri);
    }

    public static void display(SimpleDraweeView draweeView, Uri uri, int width, int height) {
        if (uri == null) {
            Log.e(TAG, "display: error the url is empty");
            return;
        }
        ImageRequest request = ImageRequestBuilder.newBuilderWithSource(uri)
                .setResizeOptions(new ResizeOptions(width, height))
                .setAutoRotateEnabled(true)
                .setLocalThumbnailPreviewsEnabled(true)
                .build();

        PipelineDraweeController controller = (PipelineDraweeController) Fresco.newDraweeControllerBuilder()
                .setOldController(draweeView.getController())
                .setImageRequest(request)
                .build();

        draweeView.setController(controller);
    }


    public static void display(SimpleDraweeView draweeView, String url, int width, int height) {
        if (TextUtils.isEmpty(url)) {
            Log.e(TAG, "display: error the url is empty");
            return;
        }
        Uri uri = Uri.parse(url);
        display(draweeView, uri, width, height);
    }

    public static void display(SimpleDraweeView draweeView, Uri uri, boolean isSmall) {
        if (uri == null) {
            Log.e(TAG, "display: error the url is empty");
            return;
        }
        ImageRequest request;
        if (isSmall) {
            request = ImageRequestBuilder.newBuilderWithSource(uri)
                    .setAutoRotateEnabled(true)
                    .setImageType(ImageRequest.ImageType.SMALL)
                    .setLocalThumbnailPreviewsEnabled(true)
                    .build();
        } else {
            request = ImageRequestBuilder.newBuilderWithSource(uri)
                    .setAutoRotateEnabled(true)
                    .setImageType(ImageRequest.ImageType.DEFAULT)
                    .setLocalThumbnailPreviewsEnabled(true)
                    .build();
        }

        PipelineDraweeController controller = (PipelineDraweeController) Fresco.newDraweeControllerBuilder()
                .setOldController(draweeView.getController())
                .setImageRequest(request)
                .build();

        draweeView.setController(controller);
    }


    public static void display(SimpleDraweeView draweeView, String url, boolean isSmall) {
        if (TextUtils.isEmpty(url)) {
            Log.e(TAG, "display: error the url is empty");
            return;
        }
        Uri uri = Uri.parse(url);
        display(draweeView, uri, isSmall);
    }

    public static void prefetchPhoto(Context context, Uri uri, int width, int height) {
        if (uri == null) {
            Log.e(TAG, "display: error the url is empty");
            return;
        }
        ImageRequest request = ImageRequestBuilder.newBuilderWithSource(uri)
                .setResizeOptions(new ResizeOptions(width, height))
                .setAutoRotateEnabled(true)
                .setRequestPriority(Priority.LOW)
                .setLocalThumbnailPreviewsEnabled(true)
                .build();

        Fresco.getImagePipeline().prefetchToDiskCache(request, context);
    }

    public static void prefetchPhoto(Context context, String url, int width, int height) {
        if (TextUtils.isEmpty(url)) {
            Log.e(TAG, "display: error the url is empty");
            return;
        }
        Uri uri = Uri.parse(url);
        prefetchPhoto(context, uri, width, height);
    }

//从磁盘缓存中获取图片文件，需要重命名成jpg文件并移动到相应储存位置
    public static File obtainCachedPhotoFile(CacheKey cacheKey) {
        File localFile = null;
        if (cacheKey != null) {
            if (ImagePipelineFactory.getInstance().getMainFileCache().hasKey(cacheKey)) {
                BinaryResource binaryResource = ImagePipelineFactory.getInstance().getMainFileCache().getResource(cacheKey);

                localFile = ((FileBinaryResource) binaryResource).getFile();
            } else if (ImagePipelineFactory.getInstance().getSmallImageFileCache().hasKey(cacheKey)) {
                BinaryResource binaryResource = ImagePipelineFactory.getInstance().getSmallImageFileCache().getResource(cacheKey);

                localFile = ((FileBinaryResource) binaryResource).getFile();
            }
        }

        return localFile;
    }

}
```
###获取Fresco磁盘缓存中的图片
```java
public static File obtainCachedPhotoFile(CacheKey cacheKey) {
    File localFile = null;
    if (cacheKey != null) {
        if (ImagePipelineFactory.getInstance().getMainFileCache().hasKey(cacheKey)) {
            BinaryResource binaryResource = ImagePipelineFactory.getInstance().getMainFileCache().getResource(cacheKey);

            localFile = ((FileBinaryResource) binaryResource).getFile();
        } else if (ImagePipelineFactory.getInstance().getSmallImageFileCache().hasKey(cacheKey)) {
            BinaryResource binaryResource = ImagePipelineFactory.getInstance().getSmallImageFileCache().getResource(cacheKey);
            localFile = ((FileBinaryResource) binaryResource).getFile();
        }
    }

    return localFile;
 }
```


###Fresco暂停与继续加载图片
```java
Fresco.getImagePipeline().resume();
Fresco.getImagePipeline().pause();
```
