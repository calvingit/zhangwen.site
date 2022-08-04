---
title: Dart 的 implements/extends/with 关键字的区别
slug: widget-key
date: 2021-03-14
categories:
  - Flutter
tags:
  - Dart
toc: false
---

主要混淆点在于 Dart 没有`Interface`或者`Protocol`关键字，但却有对应的功能，只不过全基于`class`实现。

## `implements`

在 Dart 中，每个类都可以作为接口存在，`implements`关键字可以实现多个接口，如下面的例子，类 D 实现了 A、B、C 三个接口：

```dart
class A {
  String? name;
  doA() {}
}

class B {
  doB() {}
}

class C {
  doC() {}
}

class D implements A, B, C {
  @override
  String? name;

  doD() {
    name = '';
  }

  @override
  doA() {}

  @override
  doB() {}

  @override
  doC() {}
}
```

**需要注意的是：必须`override`实现接口的所有方法或属性，Dart 中不存在某个方法是可选的情况。这个和 Swift 的 Protocol 概念有所区别。**

当然为了更好区分普通的 class 和接口 class，可以使用关键字`abstract`来区别。

```dart
abstract class X {
  doX();
}

class Y implements X {
  @override
  doX() {
    // TODO: implement doX
    return null;
  }
}
```

## `extends`

`extends`是面向对象里面的继承概念，只能单一继承

```
class A {
  doA() {
    print('A');
  }
}

class B {
  doB() {
    print('B');
  }
}

// 不允许这样
class C extends A, B {
}
```

## `with`

`with`用于`mixin`。`mixin`是只有方法的 class，它不需要继承。谁想要`mixin`的功能，使用`with`包含进来即可，可以包含多个`mixin`，有需要的话也可以`override`掉`mixin`的方法。

```dart
mixin A {
  doA () {
    //
  }
}

mixin B {
  doB () {
    //
  }
}

class C with A, B {

  @override
  doB() {
    //
  }
}
```

**普通的 class，如果没有构造函数，也可以作为 mixin**

```dart
class D {
  doD() {
    print('d');
  }
}

class E with D {

}

main() {
  var e = E();
  e.doD();
}
```

**`on`关键字可以强制 mixin 只能用于特定类型**

```dart
mixin Z on D {
  doZ() { }
}

class D {
  doD() {
    print('d');
  }
}

// 不允许
class E with Z {

}

// 允许
class F extends D with Z {

}
```

## 总结

- `implements` 带有强制性，所有的方法或属性都要`override`，但是可以实现多个接口
- `extends` 带有继承性，只能单个继承
- `mixin` 共享方法，可以多个类型共享
