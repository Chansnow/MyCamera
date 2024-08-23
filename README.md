# MyCamera
Android Camera apk（java version)

# 创建项目
## 添加 Gradle 依赖项
1. 打开 `CameraXApp.app` 模块的 `build.gradle` 文件，并添加 CameraX 依赖项：
```
dependencies {
  def camerax_version = "1.1.0-beta01"
  implementation "androidx.camera:camera-core:${camerax_version}"
  implementation "androidx.camera:camera-camera2:${camerax_version}"
  implementation "androidx.camera:camera-lifecycle:${camerax_version}"
  implementation "androidx.camera:camera-video:${camerax_version}"

  implementation "androidx.camera:camera-view:${camerax_version}"
  implementation "androidx.camera:camera-extensions:${camerax_version}"
}
```
![image](https://github.com/user-attachments/assets/f3089f28-c9fb-4abe-b987-a0523d09144e)

2. CameraX 需要一些属于 Java 8 的方法，因此我们需要相应地设置编译选项。在 `android` 代码块的末尾，紧跟在 `buildTypes` 之后，添加以下代码：
```
compileOptions {
    sourceCompatibility JavaVersion.VERSION_1_8
    targetCompatibility JavaVersion.VERSION_1_8
}
```
3. 此 Codelab 使用 ViewBinding，因此请使用以下代码（在 `android{}` 代码块末尾）启用它：
```
buildFeatures {
   viewBinding true
}
```
出现提示时，点击 Sync Now

ViewBinding ，顾名思义是“视图绑定”。它可以自动为 XML 布局文件生成一个绑定类，通过这个绑定类，你可以直接拿到布局中的View，再也不用 findViewById 的一个个去找了。
## 创建 Codelab 布局
在此 Codelab 的界面中，我们使用了以下内容：
- CameraX PreviewView（用于预览相机图片/视频）。
- 用于控制图片拍摄的标准按钮。
- 用于开始/停止视频拍摄的标准按钮。
- 用于放置 2 个按钮的垂直指南。
- 
我们将默认布局替换为以下代码，从而：
1. 打开位于 `res/layout/activity_main.xml` 的 `activity_main` 布局文件，并将其替换为以下代码。
```
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout
   xmlns:android="http://schemas.android.com/apk/res/android"
   xmlns:app="http://schemas.android.com/apk/res-auto"
   xmlns:tools="http://schemas.android.com/tools"
   android:layout_width="match_parent"
   android:layout_height="match_parent"
   tools:context=".MainActivity">

   <androidx.camera.view.PreviewView
       android:id="@+id/viewFinder"
       android:layout_width="match_parent"
       android:layout_height="match_parent" />

   <Button
       android:id="@+id/image_capture_button"
       android:layout_width="110dp"
       android:layout_height="110dp"
       android:layout_marginBottom="50dp"
       android:layout_marginEnd="50dp"
       android:elevation="2dp"
       android:text="@string/take_photo"
       app:layout_constraintBottom_toBottomOf="parent"
       app:layout_constraintLeft_toLeftOf="parent"
       app:layout_constraintEnd_toStartOf="@id/vertical_centerline" />

   <Button
       android:id="@+id/video_capture_button"
       android:layout_width="110dp"
       android:layout_height="110dp"
       android:layout_marginBottom="50dp"
       android:layout_marginStart="50dp"
       android:elevation="2dp"
       android:text="@string/start_capture"
       app:layout_constraintBottom_toBottomOf="parent"
       app:layout_constraintStart_toEndOf="@id/vertical_centerline" />

   <androidx.constraintlayout.widget.Guideline
       android:id="@+id/vertical_centerline"
       android:layout_width="wrap_content"
       android:layout_height="wrap_content"
       android:orientation="vertical"
       app:layout_constraintGuide_percent=".50" />

</androidx.constraintlayout.widget.ConstraintLayout>
```
2. 使用以下代码更新 `res/values/strings.xml` 文件
```
<resources>
   <string name="app_name">CameraXApp</string>
   <string name="take_photo">Take Photo</string>
   <string name="start_capture">Start Capture</string>
   <string name="stop_capture">Stop Capture</string>
</resources>
```
## 设置 MainActivity.java
1. 将 `MainActivity.java` 中的代码替换为以下代码，但保留软件包名称不变。它包含 import 语句、将要实例化的变量、要实现的函数以及常量。

系统已实现 `onCreate()`，供我们检查相机权限、启动相机、为照片和拍摄按钮设置 `onClickListener()`，以及实现 `cameraExecutor`。虽然系统已为您实现 `onCreate()`，但在我们实现文件中的方法之前，相机将无法正常工作。
```
package com.android.example.cameraxapp;    //根据具体改

import android.Manifest;
import android.annotation.SuppressLint;
import android.content.ContentValues;
import android.content.pm.PackageManager;
import android.os.Build;
import android.os.Bundle;
import android.provider.MediaStore;
import android.util.Log;
import android.widget.Toast;

import androidx.annotation.NonNull;
import androidx.appcompat.app.AppCompatActivity;
import androidx.camera.core.CameraSelector;
import androidx.camera.core.ImageAnalysis;
import androidx.camera.core.ImageCapture;
import androidx.camera.core.ImageCaptureException;
import androidx.camera.core.ImageProxy;
import androidx.camera.core.Preview;
import androidx.camera.lifecycle.ProcessCameraProvider;
import androidx.camera.video.MediaStoreOutputOptions;
import androidx.camera.video.Quality;
import androidx.camera.video.QualitySelector;
import androidx.camera.video.Recorder;
import androidx.camera.video.Recording;
import androidx.camera.video.VideoCapture;
import androidx.camera.video.VideoRecordEvent;
import androidx.camera.view.PreviewView;
import androidx.core.app.ActivityCompat;
import androidx.core.content.ContextCompat;

import com.google.common.util.concurrent.ListenableFuture;

import java.io.File;
import java.text.SimpleDateFormat;
import java.util.Locale;
import java.util.Objects;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class MainActivity extends AppCompatActivity {

    private ActivityMainBinding viewBinding;

    private ImageCapture imageCapture = null;

    private VideoCapture videoCapture = null;
    private Recording recording = null;

    private ExecutorService cameraExecutor;


    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        viewBinding = ActivityMainBinding.inflate(getLayoutInflater());
        setContentView(viewBinding.getRoot());

        // 请求相机权限
        if (allPermissionsGranted()) {
            startCamera();
        } else {
            ActivityCompat.requestPermissions(this, Configuration.REQUIRED_PERMISSIONS,
                    Configuration.REQUEST_CODE_PERMISSIONS);
        }

        // 设置拍照按钮监听
        viewBinding.imageCaptureButton.setOnClickListener(v -> takePhoto());
        viewBinding.videoCaptureButton.setOnClickListener(v -> captureVideo());

        cameraExecutor = Executors.newSingleThreadExecutor();


    }

    private void captureVideo() {}

    private void takePhoto() {}

    private void startCamera() {}

    private boolean allPermissionsGranted() {
        for (String permission : Configuration.REQUIRED_PERMISSIONS) {
            if (ContextCompat.checkSelfPermission(this, permission)
                    != PackageManager.PERMISSION_GRANTED) {
                return false;
            }
        }
        return true;
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        cameraExecutor.shutdown();
    }

    static class Configuration {
        public static final String TAG = "CameraxBasic";
        public static final String FILENAME_FORMAT = "yyyy-MM-dd-HH-mm-ss-SSS";
        public static final int REQUEST_CODE_PERMISSIONS = 10;
        public static final int REQUEST_AUDIO_CODE_PERMISSIONS = 12;
        public static final String[] REQUIRED_PERMISSIONS =
                Build.VERSION.SDK_INT <= Build.VERSION_CODES.P ?
                        new String[]{Manifest.permission.CAMERA,
                                Manifest.permission.RECORD_AUDIO,
                                Manifest.permission.WRITE_EXTERNAL_STORAGE} :
                        new String[]{Manifest.permission.CAMERA,
                                Manifest.permission.RECORD_AUDIO};
    }
}
```
# 请求必要的权限
应用需要获得用户授权才能打开相机；录制音频也需要麦克风权限；在 Android 9 (P) 及更低版本上，MediaStore 需要外部存储空间写入权限。在此步骤中，我们将实现这些必要的权限。
1. 打开 `AndroidManifest.xml`，然后将以下代码行添加到 `application` 标记之前。
```
<uses-feature android:name="android.hardware.camera.any" />
<uses-permission android:name="android.permission.CAMERA" />
<uses-permission android:name="android.permission.RECORD_AUDIO" />
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"
   android:maxSdkVersion="28" />
```
添加 `android.hardware.camera.any` 可确保设备配有相机。指定 `.any` 表示它可以是前置摄像头，也可以是后置摄像头。
2. 在`MainActivity`中添加以下方法下面几项内容将会详细介绍我们刚刚复制的代码。
```
@Override
public void onRequestPermissionsResult(int requestCode, @NonNull String[] permissions, @NonNull int[] grantResults) {
    super.onRequestPermissionsResult(requestCode, permissions, grantResults);
    if (requestCode == Configuration.REQUEST_CODE_PERMISSIONS) {
        if (allPermissionsGranted()) {// 申请权限通过
            startCamera();
        } else {// 申请权限失败
            Toast.makeText(this, "用户拒绝授予权限！", Toast.LENGTH_LONG).show();
            finish();
        }
    } else if (requestCode == Configuration.REQUEST_AUDIO_CODE_PERMISSIONS) {
        if (ContextCompat.checkSelfPermission(this,
                                              "Manifest.permission.RECORD_AUDIO") != PackageManager.PERMISSION_GRANTED) {
            Toast.makeText(this, "未授权录制音频权限！", Toast.LENGTH_LONG).show();
        }
    }
}
```
- 检查请求代码是否正确；否则，请忽略它。
```
if (requestCode == Configuration.REQUEST_CODE_PERMISSIONS) {

}
```
- 如果已授予权限，请调用 `startCamera()`。
```
if (allPermissionsGranted()) { 
   startCamera()
}
```
- 如果未授予权限，系统会显示一个消息框，通知用户未授予权限。
```
else {
    // 申请权限失败
    Toast.makeText(this, "用户拒绝授予权限！", Toast.LENGTH_LONG).show();
    finish();
}
```
3. 此时运行APP将首先发现系统弹出请求权限的对话框。
# 实现 Preview 预览用例
在相机应用中，取景器用于让用户预览他们拍摄的照片。我们将使用 CameraX `Preview` 类实现取景器。
如需使用 `Preview`，我们首先需要定义一个配置，然后系统会使用该配置创建用例的实例。生成的实例就是我们绑定到 CameraX 生命周期的内容。
1. 将此代码复制到 `startCamera()` 函数中。
下面几项内容将会详细介绍我们刚刚复制的代码。
```
private void startCamera() {
        // 将Camera的生命周期和Activity绑定在一起（设定生命周期所有者），这样就不用手动控制相机的启动和关闭。
        ListenableFuture<ProcessCameraProvider> cameraProviderFuture = ProcessCameraProvider.getInstance(this);

        cameraProviderFuture.addListener(() -> {
            try {
                // 将你的相机绑定到APP进程的lifecycleOwner中
                ProcessCameraProvider processCameraProvider = cameraProviderFuture.get();

                // 创建一个Preview 实例，并设置该实例的 surface 提供者（provider）。
                PreviewView viewFinder = (PreviewView)findViewById(R.id.viewFinder);
                Preview preview = new Preview.Builder()
                        .build();
                preview.setSurfaceProvider(viewBinding.viewFinder.getSurfaceProvider());

                // 选择后置摄像头作为默认摄像头
                CameraSelector cameraSelector = CameraSelector.DEFAULT_BACK_CAMERA;

                // 重新绑定用例前先解绑
                processCameraProvider.unbindAll();
                
                processCameraProvider.bindToLifecycle(MainActivity.this, cameraSelector,
                        preview);

            } catch (Exception e) {
                Log.e(Configuration.TAG, "用例绑定失败！" + e);
            }
        }, /*在主线程运行*/ContextCompat.getMainExecutor(this));

    }
```
- 创建 `ProcessCameraProvider` 的实例。这用于将相机的生命周期绑定到生命周期所有者。这消除了打开和关闭相机的任务，因为 CameraX 具有生命周期感知能力。
```ListenableFuture<ProcessCameraProvider> cameraProviderFuture = ProcessCameraProvider.getInstance(this);```
- 向 `cameraProviderFuture` 添加监听器。添加 `Runnable` 作为一个参数。我们会在稍后填写它。添加 `ContextCompat.getMainExecutor()` 作为第二个参数。这将返回一个在主线程上运行的 `Executor`。
```cameraProviderFuture.addListener(Runnable {},ContextCompat.getMainExecutor(this));```
- 在 `Runnable` 中，添加 `ProcessCameraProvider`。它用于将相机的生命周期绑定到应用进程中的 `LifecycleOwner`。
```ProcessCameraProvider processCameraProvider = cameraProviderFuture.get();```
- 初始化 `Preview` 对象，在其上调用 `build`，从取景器中获取 `Surface` 提供程序，然后在预览上进行设置。
```
PreviewView viewFinder = (PreviewView)findViewById(R.id.viewFinder);
                Preview preview = new Preview.Builder()
                        .build();
                preview.setSurfaceProvider(viewBinding.viewFinder.getSurfaceProvider());
```
- 创建 `CameraSelector` 对象，然后选择 `DEFAULT_BACK_CAMERA`。
 ```CameraSelector cameraSelector = CameraSelector.DEFAULT_BACK_CAMERA;```
- 创建一个 `try` 代码块。在此块内，确保没有任何内容绑定到 `cameraProvider`，然后将 `cameraSelector` 和预览对象绑定到 `cameraProvider`。
```
try {
   processCameraProvider.unbindAll();
   processCameraProvider.bindToLifecycle(MainActivity.this, cameraSelector,
                        preview);
}
```
- 有多种原因可能会导致此代码失败，例如应用不再获得焦点。将此代码封装到 catch 块中，以便在出现故障时记录日志。
```
catch (Exception e) {
                Log.e(Configuration.TAG, "用例绑定失败！" + e);
            }
```
2. 现在运行APP，将看到一个预览画面
# 实现ImageCapture拍照用例
其他用例与 `Preview` 非常相似。首先，我们定义一个配置对象，该对象用于实例化实际用例对象。若要拍摄照片，您需要实现 `takePhoto()` 方法，该方法会在用户按下 photo 按钮时调用。
1. 将此代码复制到 `takePhoto()` 方法中。
下面几项内容将会详细介绍我们刚刚复制的代码。
 ```
private void takePhoto() {
        // 确保imageCapture 已经被实例化, 否则程序将可能崩溃
        if (imageCapture != null) {
            // 创建一个 "MediaStore Content" 以保存图片，带时间戳是为了保证文件名唯一
            String name = new SimpleDateFormat(Configuration.FILENAME_FORMAT,
                    Locale.SIMPLIFIED_CHINESE).format(System.currentTimeMillis());
            ContentValues contentValues = new ContentValues();
            contentValues.put(MediaStore.MediaColumns.DISPLAY_NAME, name);
            contentValues.put(MediaStore.MediaColumns.MIME_TYPE, "image/jpeg");
            if (Build.VERSION.SDK_INT > Build.VERSION_CODES.P) {
                contentValues.put(MediaStore.Images.Media.RELATIVE_PATH, "Pictures/CameraX-Image");
            }

            // 创建 output option 对象，用以指定照片的输出方式。
            // 在这个对象中指定有关我们希望输出如何的方式。我们希望将输出保存在 MediaStore 中，以便其他应用可以显示它
            ImageCapture.OutputFileOptions outputFileOptions = new ImageCapture.OutputFileOptions
                    .Builder(getContentResolver(),
                        MediaStore.Images.Media.EXTERNAL_CONTENT_URI,
                        contentValues)
                    .build();

            // 设置拍照监听，用以在照片拍摄后执行takePicture（拍照）方法
            imageCapture.takePicture(outputFileOptions,
                    ContextCompat.getMainExecutor(this),
                    new ImageCapture.OnImageSavedCallback() {// 保存照片时的回调
                        @Override
                        public void onImageSaved(@NonNull ImageCapture.OutputFileResults outputFileResults) {
                            String msg = "照片捕获成功! " + outputFileResults.getSavedUri();
                            Toast.makeText(getBaseContext(), msg, Toast.LENGTH_SHORT).show();
                            Log.d(Configuration.TAG, msg);
                        }

                        @Override
                        public void onError(@NonNull ImageCaptureException exception) {
                            Log.e(Configuration.TAG, "Photo capture failed: " + exception.getMessage());// 拍摄或保存失败时
                        }
                    });
        }
    }
```
- 首先，获取对 `ImageCapture` 用例的引用。如果用例为 null，请退出函数。如果在设置图片拍摄之前点按“photo”按钮，它将为 null。如果没有 `return` 语句，应用会在该用例为 null 时崩溃。
```if (imageCapture != null){};```
- 接下来，创建用于保存图片的 MediaStore 内容值。请使用时间戳，确保 MediaStore 中的显示名是唯一的。
```
String name = new SimpleDateFormat(Configuration.FILENAME_FORMAT,
                    Locale.SIMPLIFIED_CHINESE).format(System.currentTimeMillis());
ContentValues contentValues = new ContentValues();
contentValues.put(MediaStore.MediaColumns.DISPLAY_NAME, name);
contentValues.put(MediaStore.MediaColumns.MIME_TYPE, "image/jpeg");
if (Build.VERSION.SDK_INT > Build.VERSION_CODES.P) {
    contentValues.put(MediaStore.Images.Media.RELATIVE_PATH, "Pictures/CameraX-Image");
}
```
- 创建一个 `OutputFileOptions` 对象。在该对象中，您可以指定所需的输出内容。我们希望将输出保存在 MediaStore 中，以便其他应用可以显示它，因此，请添加我们的 MediaStore 条目。
```
ImageCapture.OutputFileOptions outputFileOptions = new ImageCapture.OutputFileOptions
        .Builder(getContentResolver(),
        MediaStore.Images.Media.EXTERNAL_CONTENT_URI,
        contentValues)
        .build();
```
- 对 `imageCapture` 对象调用 `takePicture()`。传入 `outputOptions`、执行器和保存图片时使用的回调。接下来，您需要填写回调。
```
imageCapture.takePicture(outputFileOptions,
        ContextCompat.getMainExecutor(this),
        new ImageCapture.OnImageSavedCallback() {}
        );
```
- 如果图片拍摄失败或保存图片失败，请添加错误情况以记录失败。
```
@Override
public void onError(@NonNull ImageCaptureException exception) {
    Log.e(Configuration.TAG, "Photo capture failed: " + exception.getMessage());
}
```
- 如果拍摄未失败，即表示照片拍摄成功！将照片保存到我们之前创建的文件中，显示消息框，让用户知道照片已拍摄成功，并输出日志语句。
```
@Override
public void onImageSaved(@NonNull ImageCapture.OutputFileResults outputFileResults) {
    String msg = "照片捕获成功! " + outputFileResults.getSavedUri();
    Toast.makeText(getBaseContext(), msg, Toast.LENGTH_SHORT).show();
    Log.d(Configuration.TAG, msg);
}
```
2. 找到 `startCamera()` 方法，并将此代码复制到要预览的代码下方。
```
// 创建拍照所需的实例
imageCapture = new ImageCapture.Builder().build();
```
3. 最后，在 `try` 代码块中更新对 `bindToLifecycle()` 的调用，以包含新的用例：绑定拍照用例
```
processCameraProvider.bindToLifecycle(MainActivity.this, cameraSelector,
        preview,
        imageCapture/*,
        imageAnalysis*/,
        videoCapture);
```
4. 重新运行应用，然后按 Take Photo，将可以看到APP能够拍照并将照片保存。屏幕上应该会显示一个消息框，我们会在日志中看到一条消息。
# 实现 ImageAnalysis 预览帧分析用例
使用 `ImageAnalysis` 功能可让相机应用变得更加有趣。它允许定义实现 `ImageAnalysis.Analyzer` 接口的自定义类，并使用传入的相机帧调用该类。我们无需管理相机会话状态，甚至无需处理图像；与其他生命周期感知型组件一样，仅绑定到应用所需的生命周期就足够了。
1. 将此分析器添加为 `MainActivity.java`中的内部类。分析器会记录图像的平均亮度。如需创建分析器，我们会替换实现 `ImageAnalysis.Analyzer` 接口的类中的 `analyze` 函数。
```
private static class MyAnalyzer implements ImageAnalysis.Analyzer{
    @SuppressLint("UnsafeOptInUsageError")
    @Override
    public void analyze(@NonNull ImageProxy image) {
        Log.d(Configuration.TAG, "Image's stamp is " + Objects.requireNonNull(image.getImage()).getTimestamp());
        image.close();
    }
}
```
- 改写了官方教程的分析类，这个分析类就是打印每一个预览帧画面的时间戳。
在我们的类中实现 `ImageAnalysis.Analyzer` 接口后，我们只需在 `ImageAnalysis`, 中实例化一个 `LuminosityAnalyzer` 实例（与其他用例类似），并再次更新 `startCamera()` 函数，然后调用 `CameraX.bindToLifecycle()` 即可：
2. 在 `startCamera()` 方法中，将此代码添加到 `imageCapture` 代码下。
```
 // 设置预览帧分析
ImageAnalysis imageAnalysis = new ImageAnalysis.Builder()
    .build();
imageAnalysis.setAnalyzer(cameraExecutor, new MyAnalyzer());
```
3. 更新 `cameraProvider` 上的 `bindToLifecycle()` 调用，以包含 `imageAnalyzer`，绑定分析用例。
```
// 绑定用例至相机
processCameraProvider.bindToLifecycle(MainActivity.this, cameraSelector,
                                      preview,
                                      imageCapture,
                                      imageAnalysis);
```
4. 现在运行APP，并查看`Logcat`日志，可以看到输出的预览图像的时间戳信息。
# 实现 VideoCapture 视频录制用例
CameraX 在 1.1.0-alpha10 版中添加了 VideoCapture 用例，并且从那以后一直在改进。请注意，`VideoCapture` API 支持很多视频捕获功能，因此，为了使此 Codelab 易于管理，此 Codelab 仅演示如何在 `MediaStore` 中捕获视频和音频。

1. 将此代码复制到 `captureVideo()` 方法：该方法可以控制 `VideoCapture` 用例的启动和停止。下面几项内容将会详细介绍我们刚刚复制的代码。
```
@SuppressLint("CheckResult")
    private void captureVideo() {
        // 确保videoCapture 已经被实例化，否则程序可能崩溃
        if (videoCapture != null) {
            // 禁用UI，直到CameraX 完成请求
            viewBinding.videoCaptureButton.setEnabled(false);

            Recording curRecording = recording;
            if (curRecording != null) {
                // 如果正在录制，需要先停止当前的 recording session（录制会话）
                curRecording.stop();
                recording = null;
                return;
            }

            // 创建一个新的 recording session
            // 首先，创建MediaStore VideoContent对象，用以设置录像通过MediaStore的方式保存
            String name = new SimpleDateFormat(Configuration.FILENAME_FORMAT, Locale.SIMPLIFIED_CHINESE)
                    .format(System.currentTimeMillis());
            ContentValues contentValues = new ContentValues();
            contentValues.put(MediaStore.MediaColumns.DISPLAY_NAME, name);
            contentValues.put(MediaStore.MediaColumns.MIME_TYPE, "video/mp4");
            if (Build.VERSION.SDK_INT > Build.VERSION_CODES.P) {
                contentValues.put(MediaStore.Video.Media.RELATIVE_PATH, "Movies/CameraX-Video");
            }

            MediaStoreOutputOptions mediaStoreOutputOptions = new MediaStoreOutputOptions
                    .Builder(getContentResolver(), MediaStore.Video.Media.EXTERNAL_CONTENT_URI)
                    .setContentValues(contentValues)
                    .build();
            // 申请音频权限
            if (ActivityCompat.checkSelfPermission(this, Manifest.permission.RECORD_AUDIO) != PackageManager.PERMISSION_GRANTED) {
                ActivityCompat.requestPermissions(this,
                        new String[]{Manifest.permission.RECORD_AUDIO},
                        Configuration.REQUEST_CODE_PERMISSIONS);
            }
            Recorder recorder = (Recorder) videoCapture.getOutput();
            recording = recorder.prepareRecording(this, mediaStoreOutputOptions)
                        .withAudioEnabled() // 开启音频录制
                        // 开始新录制
                    .start(ContextCompat.getMainExecutor(this), videoRecordEvent -> {
                        if (videoRecordEvent instanceof VideoRecordEvent.Start) {
                            viewBinding.videoCaptureButton.setText(getString(R.string.stop_capture));
                            viewBinding.videoCaptureButton.setEnabled(true);// 启动录制时，切换按钮显示文本
                        } else if (videoRecordEvent instanceof VideoRecordEvent.Finalize) {//录制完成后，使用Toast通知
                            if (!((VideoRecordEvent.Finalize) videoRecordEvent).hasError()) {
                                String msg = "Video capture succeeded: " +
                                        ((VideoRecordEvent.Finalize) videoRecordEvent).getOutputResults()
                                        .getOutputUri();
                                Toast.makeText(getBaseContext(), msg, Toast.LENGTH_SHORT).show();
                                Log.d(Configuration.TAG, msg);
                            } else {
                                if (recording != null) {
                                    recording.close();
                                    recording = null;
                                    Log.e(Configuration.TAG, "Video capture end with error: " +
                                            ((VideoRecordEvent.Finalize) videoRecordEvent).getError());
                                }
                            }
                            viewBinding.videoCaptureButton.setText(getString(R.string.start_capture));
                            viewBinding.videoCaptureButton.setEnabled(true);
                        }
                    });
        }
    }
```
- 检查是否已创建 VideoCapture 用例：如果尚未创建，则不执行任何操作。
```if (videoCapture != null){};```
- 在 CameraX 完成请求操作之前，停用界面；在后续步骤中，它会在我们的已注册的 VideoRecordListener 内重新启用。
```viewBinding.videoCaptureButton.setEnabled(false);```
- 如果有正在进行的录制操作，请将其停止并释放当前的 `recording`。当所捕获的视频文件可供我们的应用使用时，我们会收到通知。
```
Recording curRecording = recording;
if (curRecording != null) {
    // 如果正在录制，需要先停止当前的 recording session（录制会话）curRecording.stop();
    recording = null;
    return;
}
```
- 为了开始录制，我们会创建一个新的录制会话。首先，我们创建预定的 MediaStore 视频内容对象，将系统时间戳作为显示名（以便我们可以捕获多个视频）。
```
String name = new SimpleDateFormat(Configuration.FILENAME_FORMAT, Locale.SIMPLIFIED_CHINESE)
                    .format(System.currentTimeMillis());
ContentValues contentValues = new ContentValues();
contentValues.put(MediaStore.MediaColumns.DISPLAY_NAME, name);
contentValues.put(MediaStore.MediaColumns.MIME_TYPE, "video/mp4");
if (Build.VERSION.SDK_INT > Build.VERSION_CODES.P) {
    contentValues.put(MediaStore.Video.Media.RELATIVE_PATH, "Movies/CameraX-Video");
}
```
- 使用外部内容选项创建 `MediaStoreOutputOptions.Builder`。
```
MediaStoreOutputOptions mediaStoreOutputOptions = new MediaStoreOutputOptions
        .Builder(getContentResolver(), MediaStore.Video.Media.EXTERNAL_CONTENT_URI)
```
- 将创建的视频 `contentValues` 设置为 `MediaStoreOutputOptions.Builder`，并构建我们的 `MediaStoreOutputOptions` 实例。
```
        .setContentValues(contentValues)
        .build()
```
- 将输出选项配置为 `VideoCapture<Recorder>` 的 `Recorder` 并启用录音：
```
MediaStoreOutputOptions mediaStoreOutputOptions = new MediaStoreOutputOptions
        .Builder(getContentResolver(), MediaStore.Video.Media.EXTERNAL_CONTENT_URI)
        .setContentValues(contentValues)
        .build();
```
- 在此录音中启用音频。
```
// 申请音频权限if (ActivityCompat.checkSelfPermission(this, Manifest.permission.RECORD_AUDIO) != PackageManager.PERMISSION_GRANTED) {
    ActivityCompat.requestPermissions(this,
            new String[]{Manifest.permission.RECORD_AUDIO},
            Configuration.REQUEST_CODE_PERMISSIONS);
}
```
- 启动这项新录制内容，并注册一个 lambda `VideoRecordEvent` 监听器。
```
.start(ContextCompat.getMainExecutor(this), videoRecordEvent -> {
   //lambda event listener
}
```
- 当相机设备开始请求录制时，将“Start Capture”按钮文本切换为“Stop Capture”。
```
if (videoRecordEvent instanceof VideoRecordEvent.Start) {
    viewBinding.videoCaptureButton.setText(getString(R.string.stop_capture));
    viewBinding.videoCaptureButton.setEnabled(true);// 启动录制时，切换按钮显示文本
}
```
- 完成录制后，用消息框通知用户，并将“Stop Capture”按钮切换回“Start Capture”，然后重新启用它：
```
else if (videoRecordEvent instanceof VideoRecordEvent.Finalize) {
    if (!((VideoRecordEvent.Finalize) videoRecordEvent).hasError()) {
        String msg = "Video capture succeeded: " +
                ((VideoRecordEvent.Finalize) videoRecordEvent).getOutputResults()
                        .getOutputUri();
        Toast.makeText(getBaseContext(), msg, Toast.LENGTH_SHORT).show();
        Log.d(Configuration.TAG, msg);
    } else {
        if (recording != null) {
            recording.close();
            recording = null;
            Log.e(Configuration.TAG, "Video capture end with error: " +
                    ((VideoRecordEvent.Finalize) videoRecordEvent).getError());
        }
    }
    viewBinding.videoCaptureButton.setText(getString(R.string.start_capture));
    viewBinding.videoCaptureButton.setEnabled(true);
}
```
2. 在 `startCamera()` 中，将以下代码放置在 `preview` 创建行之后。这将创建 `VideoCapture` 用例。
```
// 创建录像所需实例
Recorder recorder = new Recorder.Builder()
    .setQualitySelector(QualitySelector.from(Quality.HIGHEST))
    .build();
videoCapture = VideoCapture.withOutput(recorder);
```
3. 将 `Preview` + `VideoCapture` 用例绑定到生命周期相机。仍在 `startCamera()` 内，将 `cameraProvider.bindToLifecycle()` 调用替换为以下代码：
```
processCameraProvider.bindToLifecycle(MainActivity.this, cameraSelector,
                        preview,
                        imageCapture/*,
                        imageAnalysis*/,
                        videoCapture);
```
4. 现在运行代码，则可以录制一段视频，并在系统的图库中看到这段视频。
