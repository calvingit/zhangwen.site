---
title: 深入理解 SwiftUI 的可变容器 View
slug: swiftui-view
date: 2022-03-17
categories:
  - iOS
tags:
  - SwiftUI
toc: true
---

## 问题起源

SwiftUI 的可变容器 View 有`VStack`, `HStack`, `ZStack`, `ForEach`, `Group`等。

`@ViewBuilder` 因为是 `@resultBuilder`修饰的 struct，定义了很多静态方法`buildBlock`，这些方法可以接收一个或多个子 View，实际返回的是一个`TupleView`类型，最多接收 10 个参数。

我们可以用具体的方法来调用：

```swift
// TupleView<(Text, Text)>
let inner = ViewBuilder.buildBlock(
    Text("First"), Text("Second")
)

// TupleView<(TupleView<(Text, Text)>, Text)>
let outer = ViewBuilder.buildBlock(
    inner, Text("Third")
)

outer.border(.blue)
```

上面的每个`Text`都会包含一个蓝色的边框。进一步探索，我们把上面的`outer`放在`List`里面：

```swift
List {
    outer
}
```

结果就是每个`Text`变成 TableView 的经典 cell 样式。将`outer`换成`Group`包装，效果也会一样，会拆开每一个`Text`。但是`VStack`封装这 3 个`Text`的时候，结果就变成一个单独的 cell。

很显然，有些 View 是可以被拆解的，像 `TupleView` 和 `Group` 一样。所以有两个问题需要搞清楚：

1. 怎么像`List`一样修改 View 的每一个独立子 View？
2. 怎么实现一个像 `Group` 和 `TupleView` 的 View？

## 探索 VStack

再次深入到 SwiftUI 里面，会发现有些带下划线的隐藏 API ，比如用于打印状态变化的[\_printChanges()](https://twitter.com/luka_bernardi/status/1402045202714435585?s=21):

```swift
struct ViewBuilderDemo: View {
    @State var count: Int = 0
    var body: some View {
        if #available(iOS 15.0, *) {
            let _ = Self._printChanges()
        } else {
            // Fallback on earlier versions
        }

        Text("Count: \(count)")
        Button {
            count += 1
        } label: {
            Image(systemName: "square.and.arrow.up")
                .foregroundColor(.blue)
        }
    }
}
```

每次 Button 按下时，count +1，然后 console 打印:

```bash
ViewBuilderDemo: _count changed.
```

这对调试 View 的时候非常有用。

在我们深入到`VStack`的源码时，发现了 `_VariadicView`以及`VStack`的核心`_VStack­Layout`:

```swift
@frozen
public struct VStack<Content> : View where Content : View {
    @usableFromInline
    internal var _tree: _VariadicView.Tree<_VStackLayout, Content>

    @inlinable
    public init(alignment: HorizontalAlignment = .center, spacing: CGFloat? = nil, @ViewBuilder content: () -> Content) {
        _tree = .init(
            root: _VStackLayout(alignment: alignment, spacing: spacing), content: content())
    }

    public typealias Body = Swift.Never
}
```

接下来看看`_Variadic­View.Tree`：

```swift
public enum _VariadicView {
  @frozen
  public struct Tree<Root, Content> where Root : _VariadicView_Root {

    public var root: Root

    public var content: Content

    @inlinable
    internal init(root: Root, content: Content) {
            self.root = root
            self.content = content
        }

    @inlinable
    public init(_ root: Root, @ViewBuilder content: () -> Content) {
            self.root = root
            self.content = content()
        }
  }
}
```

`_Variadic­View.Tree`的参数是`_VariadicView_Root`以及`Content`的类型。

从前面可知`Content`是遵守`View`协议的，所以`_Variadict­View.Tree`也肯定是一样的。

```swift
extension _VariadicView.Tree : View where Root : _VariadicView_ViewRoot, Content : View {}
```

这个协议看起来像这样：

```swift
public protocol _VariadicView_ViewRoot : _VariadicView_Root {
  associatedtype Body : SwiftUI.View

  @ViewBuilder
  func body(children: _VariadicView.Children) -> Body
}
```

如果仔细点，可以发现这个有点像`Button­Style`以及类似的协议。并且`_Variadic­View.Children`顾名思义就是一系列子 View 的集合。

总结一下：

- `_Variadic­View.Tree`初始化需要`root`和`View­Builder`的返回值
- `Root` 将一系列子 View 包装成一个 View
- 如果`Tree`的`Content`遵守`View`协议，那`Tree`自身也遵守`View`协议

## 自定义容器 View

我们来写一个`List`和 `VStack`风格的容器 View，给每个子 View 之间加上 Divider。

首先，我们用`@ViewBuilder`来初始化，然后 body 使用 `_Variadic­View.Children` 实现，带上`DividedVStackLayout`:

```swift
struct DividedVStack<Content: View>: View {
    var content: Content

    init(@ViewBuilder content: () -> Content) {
        self.content = content()
    }

    var body: some View {
        _VariadicView.Tree(DividedVStackLayout()) {
            content
        }
    }
}

struct DividedVStackLayout: _VariadicView_UnaryViewRoot {
    @ViewBuilder
    func body(children: _VariadicView.Children) -> some View {
        children
    }
}
```

`_Variadic­View.Children` 还遵循协议`Random­Access­Collection`，它的`Element`不仅是`View`还是`Identifiable`。如此，可以使用 `_Variadic­View.Children` 来代替 `ForEach`。在每一个 element 之后增加 `Divider`，布局使用`VStack`。

当然，最后一个`Divider`还得忽略掉。我们可以记录一下最后的 element 的 id，ForEach 的时候判断一下即可。

代码如下：

```swift
struct DividedVStackLayout: _VariadicView_UnaryViewRoot {
    @ViewBuilder
    func body(children: _VariadicView.Children) -> some View {
        let last = children.last?.id

        VStack {
            ForEach(children) { child in
                child

                if child.id != last {
                    Divider()
                }
            }
        }
    }
}
```

当我们使用的时候，就可以输入好几个 View 了：

```swift
DividedVStack {
    Text("First")
    Text("Second")
    Text("Third")
}
```

![0WOo01](https://cdn.zhangwen.site/uPic/0WOo01.jpg)

但这里有一个问题，`DividedVStack`在使用 modifier 或 container 时候是把它当做一个整体来看待的，就像`VStack`一样。

```swift
DividedVStack {
    Text("First")
    Text("Second")
    Text("Third")
}
.border(.blue)

```

![79PAIM](https://cdn.zhangwen.site/uPic/79PAIM.jpg)

这里的蓝色边框是包在最外面的，并没有对每一个`Text`添加 border。这就是`_Variadic­View_Unary­View­Root`在起作用了，如果有一个`Tree`，它的`Root`遵守这个协议，它就会被当作单个的 View，而不是一组 View 的集合。与之对应的就是`_Variadic­View_Multi­View­Root`协议，只要遵守了这个协议，我们就可以分开`Divided­VStack`里面的每一个子 View 了。

```swift
struct Divided<Content: View>: View {
    /* … */

    var body: some View {
        _VariadicView.Tree(DividedLayout()) { content }
    }
}

struct DividedLayout: _VariadicView_MultiViewRoot {
    @ViewBuilder
    func body(children: _VariadicView.Children) -> some View {
        let last = children.last?.id

        ForEach(children) { child in
            child

            if child.id != last {
                Divider()
            }
        }
    }
}
```

写一个 demo 试试效果:

```swift
HStack {
    Divided {
        Text("First")
        Text("Second")
        Text("Third")
    }
    .border(.blue)
}
```

![](https://cdn.zhangwen.site/uPic/EuxrS2.jpg)

## 总结

Swift 是一个多范式的编程语言，除了基本的面向对象这种范式，更多的是面向 Protocol 编程。尤其是在写 SwiftUI 的时候，Protocol 贯穿了整个设计理念。

SwiftUI 的 View 是 struct 类型的，都是 immutable 的，无法使用 class 的继承方式，更做不到类似 Flutter 的 InheritedWidget 那样共享数据。

SwiftUI 布局的核心就是那些容器 View，通过深入挖掘 SDK 里面的隐藏 API，能够帮助我们更好的了解其实现细节，有助于我们自定义容器 View。
