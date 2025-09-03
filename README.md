# flutter_usb_camera

Connect to a USB UVC camera from Flutter (Android) and stream MJPEG frames.  
One method to start, one stream to receive the frames. 
Optional FPS override and **center crop** (server‑side) with MJPEG output back to Flutter as `Uint8List`.


---

## ✨ Features
- 1. Listen for camera states first:
   ```dart 
   FlutterUsbCamera.setCallbacks(
        UsbCameraCallbacks(
          onAttach: (usbDeviceInfo) => debugPrint('onAttach: ${usbDeviceInfo?.deviceName}'),
          onDeviceOpen: (usbDeviceInfo, isFirstOpen) => debugPrint('onDeviceOpen: ${usbDeviceInfo?.deviceName} '),
          onCameraOpen: (usbDeviceInfo) => debugPrint('onCameraOpen'),
          onCameraClose: (usbDeviceInfo) => debugPrint('onCameraClose'),
          onDeviceClose: (usbDeviceInfo) => debugPrint('onDeviceClose'),
          onDetach: (usbDeviceInfo) => debugPrint('onDetach'),
          onCancel: (usbDeviceInfo) => debugPrint('onCancel'),
        ),
      );
   ```
- 2. Open a USB UVC camera and stream in **MJPEG** (default 1280×720) if croppedWidth, And croppedHeight are equal 0.
   ```dart
      //Start the stream of the camera //dart
      FlutterUsbCamera.startStream(fps: 30, croppedWidth: 500, croppedHeight: 500);
    ```
- 3. Frames are delivered as **JPEG bytes** (individual MJPEG frames) — as *Unit8List*
  ```dart
   StreamSubscription<Uint8List>? _sub = FlutterUsbCamera.frames().listen((bytes) {

    });
  ```

- 4. Don't forget to call clearCallbacks() and stopStream() in dispose()
```dart
  @override
  void dispose() {
    _sub?.cancel();
    FlutterUsbCamera.clearCallbacks();
    FlutterUsbCamera.stopStream();
    super.dispose();
  }
```

> iOS is not supported. Android only.

---

## ✅ Requirements
- **Android**: API 21+ (recommended 23+)
- Real USB host device (phone/tablet that supports **USB OTG**)
- Flutter 3.x+
- Kotlin 1.9.x+ on Android side (align with your Android Gradle Plugin)

---

## 📦 Install

You can use the plugin either **locally** (path dependency) or from a **git/hosted** source.  
If your app **builds the plugin** (the usual case), you must allow the UVC dependency repository (JitPack).

### 1) Use as a local package (development)
In your Flutter **app** `pubspec.yaml`:
```yaml
dependencies:
  flutter:
    sdk: flutter
  flutter_usb_camera:
    path: ../flutter_usb_camera   # adjust the path
```

---

## 🤖 Android project setup (app/**android**)

### A) Allow JitPack
Add **JitPack** to **settings.gradle** of your Flutter app:

**`android/settings.gradle`**
```gradle
pluginManagement {
  repositories {
    google()
    mavenCentral()
    gradlePluginPortal()
    maven { url 'https://jitpack.io' }    // ✨ add
  }
}

dependencyResolutionManagement {
  repositoriesMode.set(RepositoriesMode.PREFER_SETTINGS)
  repositories {
    google()
    mavenCentral()
    maven { url 'https://jitpack.io' }    // ✨ add
  }
}

rootProject.name = "your_app_name"
include(":app")
```

> If your Gradle is older and you **don’t** have `dependencyResolutionManagement`, add `maven { url 'https://jitpack.io' }` under `allprojects.repositories { ... }` in root `build.gradle` instead.

### B) Min SDK
**`android/app/build.gradle`**
```gradle
android {
  defaultConfig {
    minSdkVersion 21   // or higher
  }
  compileOptions {
    sourceCompatibility JavaVersion.VERSION_17
    targetCompatibility JavaVersion.VERSION_17
  }
  kotlinOptions {
    jvmTarget = '"17"
  }
}
```

### C) Manifest 
Add **Camera** Permission and handle requesting in runtime.

**`android/app/src/main/AndroidManifest.xml`**
```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
          xmlns:tools="http://schemas.android.com/tools"
          package="com.example.app">

    <!-- Annonce Camera permission and handle runtime request  -->
   <uses-permission android:name="android.permission.CAMERA" />

    <application
        android:name="${applicationName}"
        android:label="Your App Name"
        android:icon="@mipmap/ic_launcher"
       
        <!-- ... -->
    </application>
</manifest>
```


### D) (Only if you see duplicate native symbols) Packaging options
Most apps don’t need this. If you hit duplicate `.so` issues, add:
```gradle
android {
  packagingOptions {
    jniLibs {
      useLegacyPackaging = true
    }
    resources {
      excludes += [
        "META-INF/LICENSE*", "META-INF/NOTICE*", "META-INF/*.kotlin_module"
      ]
    }
  }
}
```

### E) Kotlin/Gradle alignment (if you get Kotlin metadata version errors)
In **android/build.gradle** (project level), ensure Kotlin matches your AGP:
```gradle
buildscript {
  ext.kotlin_version = "1.8.22"
  dependencies {
    classpath "org.jetbrains.kotlin:kotlin-gradle-plugin:$kotlin_version"
  }
}
```
Or with newer Gradle, set it in the top‑level `plugins` block.

---

## 🧪 Quick start (Flutter) main.dart

```dart
import 'dart:async';
import 'dart:typed_data';
import 'package:flutter/material.dart';
import 'package:flutter_usb_camera/flutter_usb_camera.dart';

void main() => runApp(const MyApp());

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return const MaterialApp(home: UsbCameraPage());
  }
}

class UsbCameraPage extends StatefulWidget {
  const UsbCameraPage({super.key});

  @override
  State<UsbCameraPage> createState() => _UsbCameraPageState();
}

class _UsbCameraPageState extends State<UsbCameraPage> {
  Uint8List? _lastFrame;
  StreamSubscription<Uint8List>? _sub;

  @override
  void initState() {
    super.initState();
    // 1- Listen for camera states first
    FlutterUsbCamera.setCallbacks(
      UsbCameraCallbacks(
        onAttach: (usbDeviceInfo) => debugPrint('onAttach: ${usbDeviceInfo?.deviceName}'),
        onDeviceOpen: (usbDeviceInfo, isFirstOpen) => debugPrint('onDeviceOpen: ${usbDeviceInfo?.deviceName} '),
        onCameraOpen: (usbDeviceInfo) => debugPrint('onCameraOpen'),
        onCameraClose: (usbDeviceInfo) => debugPrint('onCameraClose'),
        onDeviceClose: (usbDeviceInfo) => debugPrint('onDeviceClose'),
        onDetach: (usbDeviceInfo) => debugPrint('onDetach'),
        onCancel: (usbDeviceInfo) => debugPrint('onCancel'),
      ),
    );

    // 2- Start: 30 FPS, center-crop to 500x500 (set to 0 to disable crop)
    FlutterUsbCamera.startStream(fps: 30, croppedWidth: 0, croppedHeight: 0);

    // 3- Get frames
    _sub = FlutterUsbCamera.frames().listen((bytes) {
      // Each event is one JPEG-encoded frame (MJPEG)
      setState(() => _lastFrame = bytes);
    });
  }

  @override
  void dispose() {
    _sub?.cancel();
    FlutterUsbCamera.clearCallbacks();
    FlutterUsbCamera.stopStream();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text("USB Camera Stream")),
      body: Center(
        child: _lastFrame == null
            ? const Text("Waiting for frames…")
            : Image.memory(
                _lastFrame!,
                gaplessPlayback: true,
                filterQuality: FilterQuality.low,
                fit: BoxFit.contain,
              ),
      ),
    );
  }
}

```

### API
```dart
/// Start USB stream
/// - fps: if > 0, try to use this fps
/// - croppedWidth/croppedHeight: if > 0, native center-crop then JPEG re-encode
FlutterUsbCamera.startStream({
  int fps = 30,
  int? croppedWidth,
  int? croppedHeight,
});

/// Subscribe to MJPEG frames (each event is a JPEG image as bytes)
Stream<Uint8List> FlutterUsbCamera.frames();

/// Stop the stream
FlutterUsbCamera.stopStream();
```

---

## 🔧 Notes & Tips

- If you see **“Manifest merger failed: application@label…”**, make sure your app’s `AndroidManifest.xml` has `xmlns:tools` at `<manifest>` and `tools:replace="android:label"` on the `<application>` element (see above).
- If you don’t see an image but you log frames and `jpeg=true`, verify you are feeding the bytes directly to `Image.memory`; no extra decoding is needed.
- Crop is **center-crop**. If you request `500×500`, you’ll get a square cut out of the center of the camera frame.
- USB permissions are requested by the UVC layer. If you see “has no permission”, unplug/replug or grant the dialog when prompted.
- Some devices negotiate MJPEG 1280×720 @ 30fps easily; others may fall back. The plugin throttles delivery on the native side to the target fps to avoid flooding Flutter.



