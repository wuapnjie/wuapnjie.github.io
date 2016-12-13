---
title: Android Camera2学习笔记
date: 2016-09-08 09:13:45
tags: [Android]
---

---
### **前言**
自从Android 5.0之后，Android有了新的Camera Api，但是现在网上的资料很少，只有谷歌的[官方示例][1]以及SDK文档，一些相关的资料，但由于想做一个相机App，所以我决定研究这个Api。

在Camera2的Api中，将一个Camera Device比作管道，输入一个个请求，返回包含一些图像的元数据和一系列的图像缓冲，Camera Device对于一系列的请求是按顺序处理。

我们可以获取的Camera Device不止一个，可能会有许多个，现在大家基本上的手机都会有2个Camera Device，一个前置的和一个后置的，如果我们还在手机上连了其他的摄像头外设，我们可以获取的Camera Device就会更多了。

那么，我们要怎么获取这些Camera Device对象呢？在Android中内置了一个CameraManager的系统级服务，我们可以这样子轻松获取

```java
CameraManager manager = (CameraManager) context.getSystemService(Context.CAMERA_SERVICE);
```

### **选择合适的相机**
每个不同的Camera Device都包含有关于这个设备的一些特性参数，比如输出图像的大小，是否支持闪光灯等信息，这些信息都通过键值对的形式储存在CameraCharacteristics对象中，这个CameraCharacteristics对象由CameraManager管理，根据每只Camera Device的Id获取

```java
for (String cameraId : cameraManager.getCameraIdList()) {
    CameraCharacteristics characteristics = cameraManager.getCameraCharacteristics(cameraId);
}
```
当获取到CameraCharacteristics对象后，我们要根据需要使用的功能选择合适的相机。
比如是否需要闪光灯支持
```java
 Boolean available = characteristics.get(CameraCharacteristics.FLASH_INFO_AVAILABLE);
mFlashSupported = available == null ? false : available;
```

是否为前置摄像头
```java
 Integer facing = characteristics.get(CameraCharacteristics.LENS_FACING);
```

以及获取图片输出的尺寸和预览画面输出的尺寸
```java
StreamConfigurationMap configurationMap = characteristics.get(CameraCharacteristics.SCALER_STREAM_CONFIGURATION_MAP);

if (configurationMap == null) continue;

//获取图片输出的尺寸
configurationMap.getOutputSizes(ImageFormat.JPEG);
//获取预览画面输出的尺寸，因为我使用TextureView作为预览
configurationMap.getOutputSizes(SurfaceTexture.class）
```

等等，都储存在CameraCharacteristics对象中，我们需要选择出符合我们条件要求的相机，并记录下相应的CameraId。

### **打开相机**
获取了CameraId后，就可以根据CameraId打开相应的Camera，获取CameraDevice对象。
```java
CameraManager manager = (CameraManager) mActivity.getSystemService(Context.CAMERA_SERVICE);
try {
    manager.openCamera(mCameraId, mStateCallback, mMainHandler);
} catch (CameraAccessException e) {
    e.printStackTrace();
}
```

查看API，openCamera需要三个参数
![image_1apnlgr2anpu19mpguodps1jjb9.png-22.7kB][2]
第一个是我们之前获取的CameraId，第二个参数是当CameraDevice被打开时的回调StateCallback，在这个回调里我们可以获取到CameraDevice对象，第三个参数是一个Handler，决定了回调函数触发的线程，若为null，则选择当前线程。

在StateCallback中，我们要实现三个方法，分别为CameraDevice被打开时触发的方法，失去连接时触发的方法，发生异常时触发的方法。在第一个方法中我们可以成功获取到CameraDevice对象，从而进行接下来的相关操作。
```java
private CameraDevice.StateCallback mStateCallback = new CameraDevice.StateCallback() {
        @Override
        public void onOpened(@NonNull CameraDevice camera) {
            //打开成功，可以获取CameraDevice对象
        }

        @Override
        public void onDisconnected(@NonNull CameraDevice camera) {
           //断开连接
        }

        @Override
        public void onError(@NonNull CameraDevice camera, final int error) {
            //发生异常
        }
    };
```

### **发送预览请求**
在发送预览请求之前，我们必须先通过CameraDevice对象创建一个CameraCaptureSession的会话对象，通过CameraCaptureSession对象发送预览请求
创建CameraCaptureSession对象同样需要三个参数，第一个是相机预览数据输出的Surface的集合，第二个是回调，在这个回调中我们可以获取CameraCaptureSession对象，第三个参数是Handler，回调触发的线程。
```java
 mCameraDevice.createCaptureSession(Arrays.asList(surface, mImageReader.getSurface()),
                    new CameraCaptureSession.StateCallback() {
                        @Override
                        public void onConfigured(@NonNull CameraCaptureSession session) {
                            mCaptureSession = session;
                            //发送预览请求
                            sendRepeatPreviewRequest();

                        }

                        @Override
                        public void onConfigureFailed(@NonNull CameraCaptureSession session) {

                        }
                    }, mCameraHandler);
```

sendRepeatPreviewRequest函数
```java
private boolean sendRepeatPreviewRequest() {
        try {
            CaptureRequest.Builder builder = mCameraDevice.createCaptureRequest(CameraDevice.TEMPLATE_PREVIEW);
            builder.addTarget(mPreviewSurface);
            builder.set(CaptureRequest.CONTROL_MODE, CameraMetadata.CONTROL_MODE_AUTO);
            builder.setTag(RequestTag.Preview);
            addBaselineCaptureKeysToRequest(builder);

            mCaptureSession.setRepeatingRequest(builder.build(),
                    mFocusStateListener,
                    mCameraHandler);
            return true;
        } catch (CameraAccessException e) {
            e.printStackTrace();
            return false;
        }
    }
```

### **发送拍摄请求**
```java
CaptureRequest.Builder builder = mCameraDevice.createCaptureRequest(CameraDevice.TEMPLATE_STILL_CAPTURE);
            builder.setTag(RequestTag.Capture);
            addBaselineCaptureKeysToRequest(builder);
            builder.set(CaptureRequest.JPEG_ORIENTATION,
                    CameraUtil.getJPEGOrientation(parameters.getOrientation(), mCameraCharacteristics));
            builder.addTarget(mPreviewSurface);
            builder.addTarget(mImageReader.getSurface());
            mCaptureSession.capture(builder.build(),
                    mFocusStateListener,
                    mCameraHandler);
```

由于Android Camera2 的API使用不是那么友好，所以大家在使用前最好学习一下相关的开源项目。
我也写了一个简单的APP，目前支持预览，点击对焦，前后摄像头切换，闪光灯模式切换，未来还会添加许多新的功能，利用Camera2 的API。

如果你有兴趣，那么和我一起来开发吧。

项目地址：[PoiCamera](https://github.com/wuapnjie/PoiCamera)


![device-2016-07-31-215411.png](http://upload-images.jianshu.io/upload_images/730559-010e5bec91c847aa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

  [1]: https://github.com/googlesamples/android-Camera2Basic
  [2]: http://static.zybuluo.com/wuapnjie/bqd1y8ziv6v7x8jkzd0xf4lo/image_1apnlgr2anpu19mpguodps1jjb9.png
