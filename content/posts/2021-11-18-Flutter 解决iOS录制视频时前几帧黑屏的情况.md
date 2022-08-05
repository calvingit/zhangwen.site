---
title: Flutter 解决iOS录制视频时前几帧黑屏的情况
slug: widget-key
date: 2021-11-18
categories:
  - Flutter
tags:
  - iOS
toc: false
---

### 问题描述

官方的[camera](https://pub.dev/packages/camera)包 在录制视频时，第一次初始化后，前 100ms 左右的视频都是黑屏，第二次使用时没有这样的情况。

Google 一轮后，大部分回答是因为音频录制需要花时间准备，需要在`startVideoRecording`前调用`prepareForVideoRecording`操作来初始化 Audio Session，但是尝试过了，没有效果。

### 解决方案

在 stackoverflow 找到一个暴力的[解决方案](https://stackoverflow.com/questions/44135223/record-video-with-avassetwriter-first-frames-are-black)，直接丢弃这 100ms 的内容。在`ios/Classes/CameraPlugin.m`文件里修改如下代码即可：

```bash
-   _videoTimeOffset = CMTimeMake(0, 1);
-    _audioTimeOffset = CMTimeMake(0, 1);
+   _videoTimeOffset = CMTimeMakeWithSeconds(1, 10);
+   _audioTimeOffset = CMTimeMakeWithSeconds(1, 10);
```

fork 了一份官方的代码，然后创建一个新的[flutter-camera](https://github.com/calvingit/flutter-camera.git)

### 如何使用

在 pub.yaml 中添加如下代码：

```yaml
dependency_overrides:
  camera:
    git:
      url: 'https://github.com/calvingit/flutter-camera.git'
      ref: 'master'
```
