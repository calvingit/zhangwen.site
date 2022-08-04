---
title: Xcode Tips
slug: xcode-tips
date: 2021-12-24
categories:
  - Other
tags:
  - Xcode
toc: true
---

## 禁用系统的无效 Log 输出

在 `Scheme`—`Arguments`—`Environment Variables`，添加`Name`为`OS_ACTIVITY_MODE`，`Value`设为`Disable`。

## 打印 App 的加载时长

包括整体加载时长，动态库加载时长等

在 `Scheme`—`Arguments`—`Environment Variables`，添加`Name`为`DYLD_PRINT_STATISTICS`，`Value`设为`YES`。

## 自动获取 Git 的提交次数当做 Build

在`Build Phases`添加脚本:

```bash
REV=`git rev-list HEAD | wc -l | awk '{print $1}'`
/usr/libexec/PlistBuddy -c "Set :CFBundleVersion ${REV}" "${TARGET_BUILD_DIR}"/${INFOPLIST_PATH}
```

## "selector not recognized" 分析

运行时有可能"selector not recognized"，主要是少了如下几类配置：

| Flags       | 位置                                       | 作用                                                                                                                                                            |
| ----------- | ------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| -ObjC       | Other Linker Flags                         | 链接静态库中所有的 Objective-C 代码到 App，库里面有分类的时候使用                                                                                               |
| -all_load   | Other Linker Flags                         | 链接静态库中所有的代码到 App 中                                                                                                                                 |
| -force_load | Other Linker Flags                         | 链接指定静态库中所有的代码到 App 中                                                                                                                             |
| YES         | Perform Single-Object Prelink (静态库使用) | 静态库中所有的对象文件合并成单一文件，只要这个单一文件有代码被使用，所有代码都被链接到 App 中                                                                   |
| 伪符号      | 源文件                                     | 在有 Category 的方法中添加一些伪符号，如空函数体的函数、可被访问的全局变量等。在需要使用 category 的方法前引用这些伪符号，这些伪符号所在的代码会被链接到 App 中 |

> 和 "-ObjC"作用类似的有以上的五种方案。可以看出，从增加 APP 代码体积来看，伪符号方案增加得最少"Perform Single-Object Prelink"、 "-force_load" 和 "–ObjC" 次之，"-all_load" 增加得最多。在开发 iOS SDK 时，为了方便使用者手动集成，最好是减少使用者需要配置的信息，所以"伪符号"方案和 "Perform Single-Object Prelink"方案是推荐的。另外，第三方 SDK 常常是闭源的，对于使用者来说，伪符号是透明的，所以从简便性角度看，推荐"Perform Single-Object Prelink"方案。但是"Perform Single-Object Prelink"不能用于巨量源码文件，比如 2000 个文件的时候，会导致编译失败。

## 快速定位方法的使用者

除了全局搜索之外还可以用以下方法定位：双击选中方法名， 使用快捷键`Control + 1`，在弹出的菜单中选中`caller`

## OC 项目新建文件时增加类名前缀

点击工程的`Target`，在右边的菜单栏`Project Document`中添加`Class Prefix`

## Open Quickly

使用快捷键`Command + Shift + O`，可以快速打开源码文件、API 文档，支持模糊搜索变量、方法。

## 获取 Swift 的每个函数的编译时间

在**Build Settings**的**Other Swift Flags**添加`-Xfrontend -debug-time-function-bodies`

## 检查内存泄漏时开启堆栈日志

有时候仅仅打开`Debug Memory Graph`进行内存泄漏调试时，有很多紫色的感叹号提示，但是信息不明确，只有一个内存地址，这时可以在`Scheme`里面打开 `Melloc Stack`，然后选中其中一个内存泄漏的地址，在右边的`Backtrace`里面有调用栈信息。
