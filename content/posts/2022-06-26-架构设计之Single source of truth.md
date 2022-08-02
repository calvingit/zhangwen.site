---
title: 架构设计之 Single source of truth
slug: ssot
date: 2022-06-26
categories:
  - Others
tags:
  - Architecture
toc: true
---

本文探讨在设计 GUI 客户端的架构模式时遇到的问题以及解决方案。

所有带图形交互的客户端都可以叫 GUI 客户端，包含但不限于：Windows/Mac/Linux 客户端，Web 页面，Android/iOS App，小程序，快应用，KFC 点餐台等各种显示屏等。

# 数据的复杂性

GUI 的架构设计说到底是设计一种处理数据的方式，GPU 展示的内容来自于 CPU 给定的不同数据。

以 iOS App 为例，因为 App 端的数据源是最多的，比如 HTTP 接口数据、TCP/MQTT 实时数据、APNS 推送数据、URL Scheme、Drag And Drop、微信分享等 App 外部数据，还有本地缓存、用户点击、键盘输入、剪贴板、跨屏传递参数等 App 内部数据。

数据源多了，必然带来各种不确定性和复杂性，如果处理不当，混乱也就在所难免，代码最后变成屎山。这就像开会一样，每个人都插一嘴，这个会议变成菜市场一样，不知道该听谁的，只听到嗡嗡作响。

## MVVM 是 GUI 架构的银弹吗？

8 年前，当我看到 MVVM 开发模式时，我以为它是解决 MVC 的“Massive Controller”问题的最佳方案。几年实践下来，事实证明我图样图森破。

MVVM 确实解决了责任不够清晰、单元测试不够友好的问题，带来了双向绑定这个概念，但是随即也产生了另一个更麻烦的问题：Massive ViewModel。项目刚开始，感觉 MVVM 很爽，随着时间推移，代码写着写着，所有的逻辑都归集到 ViewModel，View 越复杂，ViewModel 的复杂性翻倍，最后同样变成难以维护的代码屎山。

为什么会导致这个问题？

归根结底还是多数据源的复杂性带来的，MVVM 没有解决这个问题。

MVC 过渡到 MVVM，无非是把 Controller 的逻辑移到 ViewModel，ViewModel 再通过双向绑定将 View 和 ViewModel 的状态连接起来。双向绑定更多是解决本地状态的问题，比如用户输入、按钮点击、Label 实时展示等状态，少了双向绑定的 MVVM，功力减半，只是换了一个类名。

但是原先在 Controller 里面处理的各种外部数据源的部分，MVVM 没有说要怎么解决，似乎依然留在 ViewModel 里面原封不动。

所以，MVVM 就是 GUI 架构的银弹吗？我认为不是。

## Single source of truth

MVVM 的问题困扰了我 2 年左右，直到前几年，我看到 Redux 的三原则之一"Single Source of Truth"时，突然感觉醍醐灌顶，打通任督二脉，全身通泰。

> [Redux](https://redux.js.org/) 是 Web 端的状态管理库，深受[Flux](https://facebook.github.io/flux)、[CQRS](https://martinfowler.com/bliki/CQRS.html)、[Event Sourcing](https://martinfowler.com/eaaDev/EventSourcing.html) 的影响。
>
> Redux 三原则包括：
>
> - Single source of truth
> - State is read-only
> - Changes are made with pure functions

Single source of truth，简称 SSOT，中文翻译过来就是单一数据源的意思。Redux 是这么解释 SSOT 的：App 的全局状态存储在一个 tree 对象里面，而这个 tree 对象被单一的 store 持有。

当前 UI 呈现什么状态，是根据不同的数据展现的，比如已登录状态显示用户的姓名，未登录就显示登录按钮。UI 组件可以嵌套，对应的状态也可以嵌套，就像树形结构一样。App 就是这一整棵树，树根就是单一的 store，状态就是树叶、果子需要的水分。所有的状态都来源于 store，也只相信 store 的数据，无论是浇水、地下水、下雨、还是吸收空气中的水分，这些数据都通过树根这个 store 传递给树的叶子结点。

将 App 的所有全局状态归拢到一个全局对象 store 里面，任何需要用到全局状态的地方，都可以直接调用 store 的 state 即可，单一的状态来源就解决了数据源过多的问题。

把简单的问题复杂化非常容易，把复杂的问题简单化就很难了。

Redux 第二、第三个原则是相互的，首先它要求状态是只读的，然后状态的改变动作必需是纯函数。这两个原则在 JavaScript 里面非常容易实现，我们可以根据不同平台的特性酌情删减或修改，没有必要完全照搬。比如，如果需要响应式开发，那么状态可以变成一个 Observable 的状态对象。

## 状态是全局还是本地？

既然有全局状态，那必然还有本地状态，如何去确定一个状态是全局还是本地？比如下面这几个问题：

1. 某个页面里对话框的打开和关闭状态
2. 跨多个页面传递的参数
3. 当前路由
4. 当前主题样式设置、语言设置
5. 当前用户的登录状态
6. 当前用户的联系人列表

我的判断标准有两个筛选条件：

- **是否关系到整个 App？**
- **是否在大多数页面/组件中被需要**

首先，可以确定全局状态的是 4 和 5，因为这两个涉及到 App 的所有页面。路由看起来属于全局，但是它应该独立出一个组件，专门用来管理路由，所以无需放在全局状态里面，不然就重复了。

问题 1 和问题 6 肯定是本地状态了，这两个都只在当前页面里面展示。

最难判断的是问题 2，根据第二个条件，可以判断 2 可以放在全局状态中，这样在跨多个页面时无需依次传递参数给每个界面。比如微信里面的修改个人昵称，在“我”—“个人信息”—“更改名字”里面都需要展示昵称，并且在“更改名字”里面修改之后，需要在“我”的页面里面实时显示修改之后的昵称。

但是问题 2 又不符合第一个条件，如何决策呢？这就涉及到上面说到的树形结构。一棵树，有各种分支，分支里面又可以有自己独立的分支。问题 2 就是分支中的子分支，也就是状态里面的子状态。我们可以通过过滤工具，只监听 user 对象里面的 nickName 字段，无需观察整个 user。

## 单项数据流

![dEig4j](https://cdn.zhangwen.site/uPic/dEig4j.jpg)

Redux 的组件分为以下几个：

- Store: 保存数据的地方，全局对象
- State: store 里面存储的数据，store 可以存储多个 state，state 是只读的
- Action: 可以理解成对 state 的增删改操作，可以携带 payload 参数
- Reducer：这个是一个函数，参数是 Action、State，返回值是一个新的 State，里面需要实现增删改的具体逻辑
- Dispather：store 的一个函数，参数是 Action，这是改变 state 的唯一方法

上图中 state 变化时自动重新渲染 UI，这个流程不只有 React Components 支持，还有比如 Flutter、SwitUI、Jetpack Compose 等。

上图是一个单项的循环，View 只依赖 State 的变化，数据的源头只有一个 State，即单一的数据源原则。

如果要在 View 和 State 中间增加绑定连接，那么 Redux 也只能是单项绑定。

## 总结

在做架构设计时，不要盲目信奉某一个架构模式，要根据不同的平台、语言、框架、业务复杂度等因素综合考虑。架构也不是项目初期定下来就永久不变的，当业务规模增长到一定程度，无法维护时，重构就变成必然的选择。

下面我们看看 Android 这几年的架构变化：

> 1. 在 Android Java 的早期时代，默认就是 MVC 模式，Activity 承担了 Controller 的责任。
>
> 2. 为了将 Activity 中的表现逻辑彻底分离出来，业界提出了 MVP 的设计，将 UI 的表现逻辑放在 Presenter，数据处理放在 Model 层。
>
> 3. 后来 Google 推出 Jetpack，一个由多个库组成的套件，包括用于界面组件和数据源的绑定库 databinding，封装了 SQLite 的 room 等，这个时候 Google 主推 MVVM+双向绑定的架构模式。
>
> 4. 随着 Kotlin 语言变成一等公民，迎来了 Android Kotlin 时代，接着 Google 推出 Compose 的框架，带来了 MVI。不过，MVI 并不是一个全新的设计模式，其背后设计理念与 Redux 模式如出一辙，借鉴了 Redux 的思想，提倡单一数据源和单项数据流。

Android 的一生经历了四个架构模式的变化，哪个架构模式更好呢？这是没有定论的，并不是说有了 MVVM，MVP 就一无是处。当你的项目很小时，只有 HTTP 请求，无需缓存，分层太多就会导致更多的复杂性，使用 MVP 就足够了。

我们并不是说越新潮，越复杂的架构就是最好的，只有合适的架构才是最好的。但是不可否认，从 React 到 Flutter，从 MVI 到 Compose，响应式编程似乎有一统天下的趋势。未来会怎么样，我们拭目以待。

## 参考资料

- [What does the "single source of truth" mean?](https://stackoverflow.com/questions/47182888/what-does-the-single-source-of-truth-mean)
- [Redux Three Principles](https://redux.js.org/understanding/thinking-in-redux/three-principles)
- [Android Jetpack](https://developer.android.google.cn/jetpack?hl=zh-cn)
