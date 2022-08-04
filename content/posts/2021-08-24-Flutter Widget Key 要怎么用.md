---
title: Flutter Widget Key 要怎么用
slug: widget-key
date: 2021-08-24
categories:
  - Flutter
tags:
  - Widgets
toc: false
---

Key 有两种：GlobalKey，LocalKey。

GlobalKey 是整个 App 唯一的，通常用于全局 widget 的状态， 比如跨 Widget 访问状态。

LocalKey 是本地的，局部的，通常用于同级之间比较，比如列表之间增加、删除、排序等会改变顺序的操作。

LocalKey 又分为以下几种：

- ValueKey : ValueKey('String')
  > 确保 Value 将具有唯一性，比如用户的 id
  - PageStorageKey：
    > 当你有一个滑动列表，你通过某一个 Item 跳转到了一个新的页面，当你返回之前的列表页面时，你发现滑动的距离回到了顶部。这时候，给 Sliver 一个 PageStorageKey！它将能够保持 Sliver 的滚动状态。
- ObjectKey : ObjectKey(Object)
  > 确保 Object 将具有唯一性，比如 user 的 name 字段无法唯一，但是 user 对象可以确保唯一
- UniqueKey : UniqueKey()
