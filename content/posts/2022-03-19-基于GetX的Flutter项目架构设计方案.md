---
title: 基于GetX的Flutter项目架构设计方案
slug: flutter-getx
date: 2022-03-19
categories:
  - Flutter
tags:
  - Architecture
toc: true
---

## 前言

本文探讨的是一种基于 Flutter 进行全新 App 项目的开发模式，不涉及老的代码复用等问题。

关于在现有 Android 或 iOS 项目中接入 flutter 框架，这属于混合栈开发的内容，可以参考阿里巴巴的[flutter boost](https://github.com/alibaba/flutter_boost/)的方案，或者企业微信的[FlutterThrio](https://mp.weixin.qq.com/s/JdQmgQ57nWQM99JW_ueFVg)方案。

**本文假设读者已经基本熟悉 Flutter 的开发，并对主流的 Flutter 状态管理方案(Redux, Bloc, Provider 等)有一定的了解。**

## GetX 介绍

[GetX](https://github.com/jonataslaw/getx) 是 Flutter 上的一个轻量且强大的解决方案：高性能的状态管理、智能的依赖注入和便捷的路由管理。

GetX 有 3 个基本原则：

- 性能： GetX 专注于性能和最小资源消耗。GetX 打包后的 apk 占用大小和运行时的内存占用与其他状态管理插件不相上下。如果你感兴趣，这里有一个性能测试。
- 效率： GetX 的语法非常简捷，并保持了极高的性能，能极大缩短你的开发时长。
- 结构： GetX 可以将界面、逻辑、依赖和路由完全解耦，用起来更清爽，逻辑更清晰，代码更容易维护。

GetX 并不臃肿，却很轻量。如果你只使用状态管理，只有状态管理模块会被编译，其他没用到的东西都不会被编译到你的代码中。它拥有众多的功能，但这些功能都在独立的容器中，只有在使用后才会启动。

Getx 有一个庞大的生态系统，能够在 Android、iOS、Web、Mac、Linux、Windows 和你的服务器上用同样的代码运行。 通过 Get Server 可以在你的后端完全重用你在前端写的代码。

### 为什么选择 GetX

每个 App 开发者都听说过 "将界面与业务逻辑分离 "的概念，这并不是 BLoC、MVC、MVVM 的特有的，市面上的其他标准都有这个概念。

由于使用了上下文（context），这个概念在 Flutter 中往往可以得到缓解。如果你需要上下文来寻找 InheritedWidget，你需要在界面中找到它，或者通过参数传递上下文。

我特别觉得这种解决方案非常丑陋，在团队中我们总会对 View 的业务逻辑产生依赖。GetX 与标准的做法不一样，虽然它并没有完全禁止使用 StatefulWidgets、InitState 等，但它总有类似的方法，可以更干净。

控制器是有生命周期的，例如当你需要进行 API 请求时，你不依赖于界面中的任何东西。你可以使用 onInit 来启动 http 调用，当数据到达时，变量将被填充。由于 GetX 是完全响应式的，一旦项目被填充，所有使用该变量的 widgets 将在界面中自动更新。这使得具有 UI 专业知识的人只需要处理 widget，除了用户事件（比如点击按钮）之外，不需要向业务逻辑发送任何东西，而处理业务逻辑的人将可以自由地单独创建和测试业务逻辑。

## 主要功能

App 开发有三大功能：状态管理、路由管理、依赖管理，GetX 都有对应的解决方案，还有多语言、主题切换等。

### 状态管理

目前，Flutter 有几种状态管理器。但是，它们中的大多数都涉及到使用 ChangeNotifier 来更新 widget，这对于中大型应用的性能来说是一个很糟糕的方法。你可以在 Flutter 的官方文档中查看到，ChangeNotifier 应该使用 1 个或最多 2 个监听器，这使得它们实际上无法用于任何中等或大型应用。

Get 并不是比任何其他状态管理器更好或更差，而是说你应该分析这些要点以及下面的要点来选择只用 Get，还是与其他状态管理器结合使用。

Get 有两个不同的状态管理器：简单的状态管理器（GetBuilder）和响应式状态管理器（GetX）。

**响应式状态管理器**

响应式编程可能会让很多人感到陌生，因为觉得它很复杂，但是 GetX 将响应式编程变得非常简单。

- 你不需要创建 StreamControllers.
- 你不需要为每个变量创建一个 StreamBuilder。
- 你不需要为每个状态创建一个类。
- 你不需要为一个初始值创建一个 get。

让我们想象一下，你有一个名称变量，并且希望每次你改变它时，所有使用它的小组件都会自动刷新。
这就是你的计数变量。

```dart
var name = 'Jack';
```

要想让它变得可观察，你只需要在它的末尾加上".obs"。

```dart
var name = 'Jack'.obs;
```

而在 UI 中，当你想显示该值并在值变化时更新页面，只需这样做。

```dart
Obx(() => Text("${controller.name}"));
```

这就是全部，就这么简单。

### 路由管理

导航到新页面

```dart
Get.to(NextScreen()); // class方式
Get.toNamed('/details');// URL方式
```

返回前一个页面:

```dart
Get.back(); // 退出或关闭
Get.off(NextScreen()); // 进入下一个页面，但没有返回上一个页面的选项（用于闪屏页，登录页面等）
Get.offAll(NextScreen()); // 进入下一个页面并取消之前的所有路由（在购物车、投票和测试中很有用）
```

### 依赖管理

Get 有一个简单而强大的依赖管理器，它允许你只用 1 行代码就能检索到与你的 Bloc 或 Controller 相同的类，无需 Provider context，无需 inheritedWidget。

```dart
Controller controller = Get.put(Controller()); // 而不是 Controller controller = Controller();
```

- 注意：如果你使用的是 Get 的状态管理器，请多注意绑定 api，这将使你的界面更容易连接到你的控制器。

你是在 Get 实例中实例化它，而不是在你使用的类中实例化你的类，这将使它在整个 App 中可用。
所以你可以正常使用你的控制器（或类 Bloc）。

**提示：** Get 依赖管理与包的其他部分是解耦的，所以如果你的应用已经使用了一个状态管理器（任何一个，都没关系），你不需要全部重写，你可以使用这个依赖注入。

```dart
controller.fetchApi();
```

想象一下，你已经浏览了无数条路由，现在你需要拿到一个被遗留在控制器中的数据，那你需要一个状态管理器与 Provider 或 Get_it 一起使用来拿到它，对吗？用 Get 则不然，Get 会自动为你的控制器找到你想要的数据，而你甚至不需要任何额外的依赖关系。

```dart
Controller controller = Get.find();
//是的，它看起来像魔术，Get会找到你的控制器，并将其提供给你。你可以实例化100万个控制器，Get总会给你正确的控制器。
```

然后你就可以恢复你在后面获得的控制器数据。

```dart
Text(controller.textFromApi);
```

## 项目结构

### 分层架构

按照传统分层方式划分，如下图：

<img src="https://cdn.zhangwen.site/uPic/c7duxJVfkQqyK3m.png" style="zoom:50%;" />

主要分为四个层次

- 页面层：即业务逻辑
- 组件层：每个页面通用的功能组件
- 服务层：不依赖于业务的核心功能模块
- 框架层：即 Flutter 的两个 UI 框架

### 数据流程

<img src="https://cdn.zhangwen.site/uPic/nopL6XWe24tESlw.png" style="zoom:50%;" />

Data 的来源有两个：

- Cache 本地缓存直接读取
- Repository-Provider 网络数据

**页面内的数据流转**

页面代码拆分也是 MVC 和响应式的设计思想，拆分成 Controller、View、State。

<img src="https://cdn.zhangwen.site/uPic/8IUgk3NK5lLerQi.png" alt="image-20210930112635832" style="zoom:50%;" />

## 总结

综上所述，此架构模式基本上可以做到模板化，有充分的解耦性和扩展性，但又不会增加太多复杂性，尤其适合在中小型 App 中应用。

**架构的目的就是为了解决复杂性**

当你的项目团队只有几个人的情况下，过度关注组件化、代码隔离会让团队的心智负担飙升，过于分工并不能带来很好的效果。每一种架构最终都是一个平衡的结果，一定要结合团队实际情况进行取舍，不能一味的追求高大上。
