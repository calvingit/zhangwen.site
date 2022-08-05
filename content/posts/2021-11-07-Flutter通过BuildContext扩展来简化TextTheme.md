---
title: Flutter通过BuildContext扩展来简化TextTheme
slug: flutter-texttheme
date: 2021-11-07
categories:
  - Flutter
tags:
  - Widgets
toc: false
---

## 通过 BuildContext 扩展来简化 TextTheme

`ThemeData`里面有三种文本主题：

- textTheme：默认主题，文本的颜色与卡片和画布的颜色形成对比
- primaryTextTheme：与 primaryColor 形成对比的文本主题
- accentTextTheme：与 accentColor 形成对比的文本主题

### 普通用法

通常访问主题的文本样式是这样的：

```dart
Text(
  'The Lord of the Rings',
  style: Theme.of(context).accentTextTheme.headline1,
)
```

如果你修改部分主题，则可以使用`copyWith`：

```dart
Theme.of(context).textTheme.headline2.copyWith(color: Colors.red)
```

### 简化用法

扩展`BuildContext`，增加文本主题样式：

```dart
import 'package:flutter/material.dart';


extension UIThemeExtension on BuildContext {

  // * (default) TextTheme
  TextStyle get h1 => Theme.of(this).textTheme.headline1;
  TextStyle get h2 => Theme.of(this).textTheme.headline2;
  TextStyle get h3 => Theme.of(this).textTheme.headline3;
  TextStyle get h4 => Theme.of(this).textTheme.headline4;
  TextStyle get h5 => Theme.of(this).textTheme.headline5;
  TextStyle get h6 => Theme.of(this).textTheme.headline6;

  TextStyle get sub1 => Theme.of(this).textTheme.subtitle1;
  TextStyle get sub2 => Theme.of(this).textTheme.subtitle2;

  TextStyle get body1 => Theme.of(this).textTheme.bodyText1;
  TextStyle get body2 => Theme.of(this).textTheme.bodyText2;

  TextStyle get button => Theme.of(this).textTheme.button;

  // * PrimaryTextTheme
  TextStyle get pH1 => Theme.of(this).primaryTextTheme.headline1;
  TextStyle get pH2 => Theme.of(this).primaryTextTheme.headline2;
  TextStyle get pH3 => Theme.of(this).primaryTextTheme.headline3;
  TextStyle get pH4 => Theme.of(this).primaryTextTheme.headline4;
  TextStyle get pH5 => Theme.of(this).primaryTextTheme.headline5;
  TextStyle get pH6 => Theme.of(this).primaryTextTheme.headline6;

  TextStyle get pSub1 => Theme.of(this).primaryTextTheme.subtitle1;
  TextStyle get pSub2 => Theme.of(this).primaryTextTheme.subtitle2;

  TextStyle get pBody1 => Theme.of(this).primaryTextTheme.bodyText1;
  TextStyle get pBody2 => Theme.of(this).primaryTextTheme.bodyText2;

  TextStyle get pButton => Theme.of(this).primaryTextTheme.button;

  // * AccentTextTheme
  TextStyle get aH1 => Theme.of(this).accentTextTheme.headline1;
  TextStyle get aH2 => Theme.of(this).accentTextTheme.headline2;
  TextStyle get aH3 => Theme.of(this).accentTextTheme.headline3;
  TextStyle get aH4 => Theme.of(this).accentTextTheme.headline4;
  TextStyle get aH5 => Theme.of(this).accentTextTheme.headline5;
  TextStyle get aH6 => Theme.of(this).accentTextTheme.headline6;

  TextStyle get aSub1 => Theme.of(this).accentTextTheme.subtitle1;
  TextStyle get aSub2 => Theme.of(this).accentTextTheme.subtitle2;

  TextStyle get aBody1 => Theme.of(this).accentTextTheme.bodyText1;
  TextStyle get aBody2 => Theme.of(this).accentTextTheme.bodyText2;

  TextStyle get aButton => Theme.of(this).accentTextTheme.button;
}
```

然后我们就可以像这样使用主题了：

```dart
return Container(
      padding: const EdgeInsets.all(15),
      decoration: BoxDecoration(
        color: Colors.black,
      ),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Text(
            'The Lord of the Rings: The Fellowship of the Ring',
            style: context.aH1
          ),

          Text('released on: 18/01/202', style: context.sub2),

          Image.network('https://www.themoviedb.org/t/p/original/oiwc338EoBgS4sEI2ixAny4KQKg.jpg'),

          Text('Description', style: context.h3 ),

          Text(
            'One ring to rule them all',
            style: context.body1.copyWith(color: Colors.brown),
          ),

          Text('Young hobbit Frodo Baggins, after inheriting a mysterious ring from his uncle Bilbo, must leave his home in order to keep it from falling into the hands of its evil creator. Along the way, a fellowship is formed to protect the ringbearer and make sure that the ring arrives at its final destination: Mt. Doom, the only place where it can be destroyed.',
            style: context.body1
          )
        ]
      )
    );
```
