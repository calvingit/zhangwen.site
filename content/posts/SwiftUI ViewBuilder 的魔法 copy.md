---
title: SwiftUI ViewBuilder 的魔法222
slug: viewbuilder2
date: 2021-03-08
toc: true
series: [Reading]
---

## 定义

先看一下`ViewBuilder`的定义，实际上这是一个`@resultBuilder` 的 struct。

```swift
@resultBuilder public struct ViewBuilder {

    public static func buildBlock() -> EmptyView

    public static func buildBlock<Content>(_ content: Content) -> Content where Content : View
}
```

`@resultBuilder` 属性封装具体用法查看[官方文档](https://docs.swift.org/swift-book/LanguageGuide/AdvancedOperators.html#ID630)。

## 用于函数参数的用法

下面是一个简单的例子，将 `@ViewBuilder` 用于参数

```swift
func contextMenu<MenuItems: View>(@ViewBuilder menuItems: () -> MenuItems) -> some View
```

在调用的时候可以指定多个 View，而且不需要逗号分割，

```swift
myView.contextMenu {
    Text("Cut")
    Text("Copy")
    Text("Paste")
    if isSymbol {
        Text("Jump to Definition")
    }
}
```

多个`Text`是因为 buildBlock 的多参数重载实现，最多到 C9：

```swift
static func buildBlock<C0, C1>(C0, C1) -> TupleView<(C0, C1)>
static func buildBlock<C0, C1, C2>(C0, C1, C2) -> TupleView<(C0, C1, C2)>
static func buildBlock<C0, C1, C2, C3>(C0, C1, C2, C3) -> TupleView<(C0, C1, C2, C3)>
static func buildBlock<C0, C1, C2, C3, C4>(C0, C1, C2, C3, C4) -> TupleView<(C0, C1, C2, C3, C4)>
static func buildBlock<C0, C1, C2, C3, C4, C5>(C0, C1, C2, C3, C4, C5) -> TupleView<(C0, C1, C2, C3, C4, C5)>
static func buildBlock<C0, C1, C2, C3, C4, C5, C6>(C0, C1, C2, C3, C4, C5, C6) -> TupleView<(C0, C1, C2, C3, C4, C5, C6)>
static func buildBlock<C0, C1, C2, C3, C4, C5, C6, C7>(C0, C1, C2, C3, C4, C5, C6, C7) -> TupleView<(C0, C1, C2, C3, C4, C5, C6, C7)>
static func buildBlock<C0, C1, C2, C3, C4, C5, C6, C7, C8>(C0, C1, C2, C3, C4, C5, C6, C7, C8) -> TupleView<(C0, C1, C2, C3, C4, C5, C6, C7, C8)>
static func buildBlock<C0, C1, C2, C3, C4, C5, C6, C7, C8, C9>(C0, C1, C2, C3, C4, C5, C6, C7, C8, C9) -> TupleView<(C0, C1, C2, C3, C4, C5, C6, C7, C8, C9)>
```

上面的`if`支持是因为`ViewBuilder`实现了`buildEither(second:)` 静态方法，还有其他更多的写法，比如`For`循环。

## 用于返回值的用法

先来梳理一下问题，当你创建一个函数，返回类型是`View`时，如果编译器不能在编译阶段就确定类型，那就会出现泛型无法推断类型的编译错误。

比如下面的例子，只能在运行期才能确定返回值类型。

```swift
func showTextOrImage(isImage: Bool) -> some View {

    if !isImage {
        Text("This is a title")
            .foregroundColor(.red)
    }
    else {
        Image(systemName: "square.and.arrow.up")
            .foregroundColor(.blue)
    }
}

```

有几种方式解决这个问题，核心就是再包一层，比如容易想到的就是自定义一个 View:

```swift
struct ShowTextOrImage: View {
    let isImage: Bool

    var body: some View {
        if !isImage {
            Text("This is a title")
                .foregroundColor(.red)
        }
        else {
            Image(systemName: "square.and.arrow.up")
                .foregroundColor(.blue)
        }
    }
}
```

这种方式不好的地方就是需要另写一个 struct，更好的方式是在 struct 内部通过函数就可以得到需要的 View，我们可以使用`Group`来实现：

```swift
// 使用 Group 包装以下
func groupDemo(isImage: Bool) -> some View {
    Group {
        if !isImage {
            Text("This is a title")
                .foregroundColor(.red)
        }
        else {
            return AnyView(Image(systemName: "square.and.arrow.up")
                .foregroundColor(.blue))
        }
    }
}
```

或者 转成`AnyView`擦除类型具体的类型：

```swift
// AnyView 擦除类型
func anyViewDemo(isImage: Bool) -> some View {
    if !isImage {
        return AnyView(Text("This is a title")
            .foregroundColor(.red))
    }
    else {
        return AnyView(Image(systemName: "square.and.arrow.up")
            .foregroundColor(.blue))
    }
}
```

最后一种方式就是使用 `@ViewBuilder` 属性封装，也可以达到目的。

```swift
@ViewBuilder
func viewBuilderDemo(isImage: Bool) -> some View {
    if !isImage {
        Text("This is a title")
            .foregroundColor(.red)
    }
    else {
        Image(systemName: "square.and.arrow.up")
            .foregroundColor(.blue)
    }
}
```

这里不会报错的原因，也是`@resultBuilder`的作用，因为`ViewBuilder`实现了`buildEither(second:)`，支持 if-else 语法

## 用于属性

当你想实现一个自定义的`VStack`时，可以这么做：

```swift
struct CustomVStack<Content: View>: View {
    let content: () -> Content

    var body: some View {
        VStack {
            // custom stuff here
            content()
        }
    }
}
```

但是这种方式只能接收单个`View`，无法传入多个 View：

```swift
CustomVStack {
    Text("Hello")
    Text("Hello")
}
```

为了达到原生`VStack`的效果，就必须增加一个构造函数:

```swift
init(@ViewBuilder content: @escaping () -> Content) {
    self.content = content
}
```

每次定义容器 View 时，都得这么写的话就很啰嗦，所以有人向官方提[建议](https://bugs.swift.org/browse/SR-13188)，看是否能把`@ViewBuilder`直接用于属性。

最终这个[提案通过](https://github.com/apple/swift/pull/34097)了，发布在 Swift 5.4 版本：

```swift
struct CustomVStack<Content: View>: View {
    @ViewBuilder let content: Content

    var body: some View {
        VStack {
            content
        }
    }
}
```

## 其他

- 官方文档：<https://docs.swift.org/swift-book/ReferenceManual/Attributes.html#ID633>

一些开源的`@resultBuilder`实现:

- [awesome-result-builders](https://github.com/carson-katri/awesome-result-builders)
