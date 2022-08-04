---
title: 编译 iOS 版本的 WebRTC
slug: build-ios-webrtc
date: 2021-06-07
categories:
  - Other
tags:
  - Tools
toc: true
---

iOS 平台的官方编译说明：https://webrtc.googlesource.com/src/+/main/docs/native-code/ios/index.md

## 获取源码

```bash
fetch --nohooks webrtc_ios
```

一定要确保上面的命令成功，如果出错，需要删除所有文件重来。大部分是网络问题，实在不行用代理。

PS: 这个命令会自动执行`gclient sync`

## 生成项目文件

```bash
gn gen out/ios_64 --args='target_os="ios" target_cpu="arm64"'
```

默认是 debug 模式，如果要 Release 模式，增加`is_debug=false`：

```bash
gn gen out/ios_64 --args='target_os="ios" target_cpu="arm64"' is_debug=false
```

这里可能会出现 code signing 的问题:

```bash
Automatic code signing identity selection was enabled but could
not find exactly one code signing identity matching
iPhone Developer. Check that your keychain
is accessible and that there is a valid code signing identity
listed by `xcrun security find-identity -v -p codesigning`
TIP: Simulator builds don't require code signing...
ERROR at //build/config/ios/ios_sdk.gni:142:7: Assertion failed.
      assert(false)
```

增加一个`ios_enable_code_signing=false`参数，禁用掉 code signing

```bash
gn gen out/ios_64 --args='target_os="ios" target_cpu="arm64" ios_enable_code_signing=false' is_debug=false
```

## 编译

```bash
ninja -C out/ios_64 framework_objc
```

可能出现下面的情况：

```bash
ninja: Entering directory `out/ios_64'
[1/5] COPY_BUNDLE_DATA gen/sdk/framework_objc_info_plist.plist WebRTC.framework/Info.plist
FAILED: WebRTC.framework/Info.plist
rm -rf WebRTC.framework/Info.plist && /bin/cp -Rc gen/sdk/framework_objc_info_plist.plist WebRTC.framework/Info.plist
cp: WebRTC.framework/Info.plist: clonefile failed: Operation not supported
ninja: build stopped: subcommand failed.
```

这个问题是因为`cp`命令不能包含`c`参数，通过搜索"&& /bin/cp -Rc"找到位置，删除`c`即可，重新允许编译命令即可。

下载的源码库很大，整个编译过程就相当长时间，差不多花费一个小时。
