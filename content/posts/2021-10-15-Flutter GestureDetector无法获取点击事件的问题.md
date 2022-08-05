---
title: Flutter GestureDetector无法获取点击事件的问题
slug: gestureDetector
date: 2021-10-15
categories:
  - Flutter
tags:
  - Widgets
toc: false
---

## 有哪些情况

在有`TextField`的表单界面中，点击空白部分隐藏键盘是基本功能。但是如果`GestureDetector`直接包裹着`TextField`是无法响应`onTap`事件的，比如下面这种情况：

```dart
/// 1
GestureDetector(
        onTap: () => FocusManager.instance.primaryFocus?.unfocus(),
        child: TextField(),
      )
```

即便再给`TextField`套一层`Container`，也是没有响应：

```dart
/// 2
GestureDetector(
        onTap: () => FocusManager.instance.primaryFocus?.unfocus(),
        child: Container(
          height: 300,
          color: Colors.grey[200],
          child: TextField(),
        ),
      ),
```

再或者嵌套一个`Column`：

```dart
/// 3
GestureDetector(
        onTap: () => FocusManager.instance.primaryFocus?.unfocus(),
        child: Column(
          children: [
            TextField(),
            Spacer(),
          ],
        ),
      )
```

## 解决办法

- **方案一**
  在`Column`套一层`Container`，而且必须设置颜色才会生效。

```dart
Container(
          color: Colors.red,// 必须设置颜色
          child: Column(
            children: [
              TextField(),
              Spacer(),
            ],
          ),
        ),
```

- **方案二**
  使用`ListView`包含`TextField`

```dart
ListView(
          children: [
            TextField(),
          ],
        ),
```

## 什么原因

这个问题有人在 flutter issues 提过：[GestureDetector onTap doesn't work with Column widget](https://github.com/flutter/flutter/issues/51621)，相关问题和解答可以看看。

下面来聊聊具体的东西。

`GestureDetector`有个`behavior`字段，这是个枚举类型：

```dart
/// How to behave during hit tests.
enum HitTestBehavior {
  /// Targets that defer to their children receive events within their bounds
  /// only if one of their children is hit by the hit test.
  deferToChild,

  /// Opaque targets can be hit by hit tests, causing them to both receive
  /// events within their bounds and prevent targets visually behind them from
  /// also receiving events.
  opaque,

  /// Translucent targets both receive events within their bounds and permit
  /// targets visually behind them to also receive events.
  translucent,
}
```

这三种点击事件的作用就是：

- deferToChild：child 处理事件，默认
- opaque：自己处理事件
- translucent：自己和 child 都可以接收事件

> **`Container`有个隐藏彩蛋，即当给 `Container`设置`color`的时候，点击有响应。而`color`为空，则点击无响应。具体原因查看: [Flutter : 关于 HitTestBehavior](https://blog.csdn.net/u013066292/article/details/117284085)**。

基于上面两个前提条件，我们可以得出以下情况：

```dart
// 不响应onTap
GestureDetector(
  behavior: HitTestBehavior.deferToChild, // 1
  onTap: () => print('hit'),
  child: Container(
    height: 300,
    //color: Colors.red, // 2
  ),
)
```

```dart
// 响应onTap
GestureDetector(
  behavior: HitTestBehavior.opaque, // 1
  onTap: () => print('hit'),
  child: Container(
    height: 300,
    //color: Colors.red, // 2
  ),
)
```

```dart
// 响应onTap
GestureDetector(
  behavior: HitTestBehavior.deferToChild, // 1
  onTap: () => print('hit'),
  child: Container(
    height: 300,
    color: Colors.red, // 2
  ),
)
```
