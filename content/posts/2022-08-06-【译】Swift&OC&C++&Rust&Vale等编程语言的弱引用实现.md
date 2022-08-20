---
title: 【译】Swift OC C++ Rust Vale 等编程语言的弱引用实现
slug: surprising-weak-refs
date: 2022-08-06
draft: true
categories:
  - Others
toc: true
---

> 本文翻译自 [Vale](https://vale.dev/) 核心团队的 [Blog](https://verdagon.dev/blog/surprising-weak-refs)，Vale 是一门全新的编程语言，特点是快速、安全、简单。

弱引用本身就很奇怪，本文是收集了一些常见的编程语言对弱引用的实现方法。我们的目标是找到最佳方法用于 Vale，以符合其快速、内存安全和易于使用的目标。最后，我们制定了一个全新方法。

## 弱引用有什么用？

大部分共享所有权的编程语言中，如 Python、Swift、Obj-C、C#，正常的引用是"strong"的，即我们常说的强引用。只要还有一个强引用指向对象，那么对象就不能被释放。当最后一个强引用消失之后，对象被释放。

> 在引用计数的语言里，如 Swift，当不再有强引用指向对象时，能够立即通知删除对象。在跟踪垃圾回收的语言里，如 Java，解释器会通过事件的方式通知删除对象。

相对于强引用，我们还有弱引用，类似于一个快捷方式或者软链接一样，不会对对象进行保活。目标对象被释放时，他们会变成 null。

我们知道强引用容易触发“循环引用”的问题，导致对象无法被释放，最终内存泄漏。这时我们可以引入弱引用打破这个循环，例如 Swift 里面的`[weak self]`。我们还可以通过弱引用来判断一个变量是否还存在，如果存在还可以继续后面的逻辑。

举个栗子，我们在一个 observer 里面使用：

```vale
func handleClick(self &Button) {
  // 检查 self.clickObserverWeakRef 指向的对象是否还存在.
  // 如果存在，让它指向 clickObserver.
  if clickObserver = self.clickObserverWeakRef.lock() {
    // 这里我们知道它一定存在，然后调用它的 onClick
    clickObserver.onClick();
  }
}
```

或者当做一个魔法导弹来检查它的敌人是否活着：

```vale
func takeAction(self &Missile) {
  // 检查 self.enemyWeakRef 指向的敌人是否活着.
  // 如果活着，让它指向 enemy.
  if enemy = self.enemyWeakRef.lock() {
    // 这里我们知道敌人肯定活着, 走近一步.
    self.position.goToward(enemy.position);
    if self.position == enemy.position {
      // 我们到达敌方根据地，爆炸!
      self.explode();
    }
  } else {
    // 敌人不见了，原地爆炸
    self.explode();
  }
}
```

这是个非常有用的工具，而且很简单，除了在 Objective-C 里......

## Objective-C 的全局弱指针跟踪管理

Objective-C 有一个互斥保护的全局哈希表`weak_table_t weak_table`，这里跟踪着所有的弱引用，以及他们指向的对象。

我们来看下[objc 的 源码](https://opensource.apple.com/tarballs/objc4/)，本文下载的版本是 objc4-781。

源码里有一个`SideTable`结构体：

```
struct SideTable {
    // 自旋锁
    spinlock_t slock;
    // 引用计数的hash表
    RefcountMap refcnts;
    // 存储对象弱引用指针的hash表（重点）
    weak_table_t weak_table;

    ...
}
```

`weak_table_t` 定义:

```c
/**
 * The global weak references table. Stores object ids as keys,
 * and weak_entry_t structs as their values.
 */
struct weak_table_t {
    weak_entry_t *weak_entries;
    size_t    num_entries;
    uintptr_t mask;
    uintptr_t max_hash_displacement;
};

```

`weak_entry_t` 定义：

```c
// weak表项
struct weak_entry_t {
    DisguisedPtr<objc_object> referent; // 被引用对象的指针
    weak_referrer_t  inline_referrers[WEAK_INLINE_COUNT]; // weak变量的指针数组
};
```

具体实现规则：

- 当我们创建一个新的弱引用指向对象时，我们就往`weak_table`添加一条记录`weak_entry_t`类型的 entry
  - key 是对象的地址
  - value 是弱引用自身的地址
- 当我们不需要弱引用时，直接从哈希表里移除 entry
- 当对象被释放时，找出这个对象关联的所有 entry，从 entry 取出弱引用的指针地址，让其指向 nil

这种实现方法有很多好处：

1. 给定一个弱引用，检查它是否指向一个“活着”的对象是非常快捷的，只需要检查指针是否 nil 即可。
2. 不会出现 Swift 和 Rust 都有的"zombie allocation"，减少内存占用。
3. 在创建第一个弱引用之前，它都是零开销。

当然也有劣势：

1. 创建和销毁弱引用的时候非常慢，因为它有一个全局锁，加上一些哈希表的操作。
2. 每一个弱引用占用 16 位，有时更多

更详细的可以查看这篇文章：[Objective-C weak 关键字实现源码解析](https://www.jianshu.com/p/2c2da28a5a53)。

## Swift 的僵尸对象

Swift 是个新语言，一直在快速改进中，对弱引用有两个版本实现，以 Swift 4 为分界线。

**在 Swift 4 之前**

Swift 对象有两个引用计数：强引用计数，弱引用计数。

“当最后一个强引用消失之后，对象被释放。” 这句话放在这个版本的 Swift 身上不成立！

举个栗子，Swift 的对象有两个引用，分别是强引用和弱引用，当强引用消失之后，对象被`deinit`了，但是内存没有释放。在我的理解里，这个`deinit`操作就是用`memset`全部设为 0 了。这个对象处于“僵尸状态”，没有什么有用的内容，它的存在只是因为还有一个弱引用指向它。

假如这个时候，我们去检查弱引用是否还指向一个“活着”的对象，它会检查这个对象是否强引用计数是否大于 0，如果是，则返回`true`，否则返回`false`。

当最后一个弱引用也消失之后，弱引用计数归零，对象才被释放，当然僵尸内存也被销毁。

Swift 的这种实现方式有个很大的好处：它非常快，因为引用计数是在对象的其他字段旁边，这对缓存非常有利。因为 CPU 在读取引用计数时，会很自然的将附近区域的字段读进缓存里面，这让接下来的访问操作更快。

劣势就是：

1. 僵尸对象
2. 多了一个引用计数的内存开销，即使弱引用完全没用到
3. 所有的对象都必须在堆中单独分配，我们永远不可能在一个对象的内存中看到指向别的对象的弱引用。

这种实现相对 Objective-C 来说更简单，逻辑清晰。

**在 Swift 4 之后**

https://www.mikeash.com/pyblog/friday-qa-2017-09-22-swift-4-weak-references.html

新版本的 Swift 对弱引用的新实现引入了 Side Table 的概念。

Side Table 是一个独立的内存块，存储对象的额外信息。Side Table 是可选的，这意味着一个对象没有，没有的话也就不占用内存了。

每个对象都有一个指向其 Side Table 的指针，而 Side Table 有一个指向对象的指针。Side Table 可以存储其他信息，比如对象相关的数据。

为了避免为 Side Table 保留八个字节，Swift 做了一个巧妙的优化。最初，对象的第一个字段是 class，下一个字段存储引用计数。当一个对象需要一个 Side Table 时，第二个字段被替换成一个 Side Table 指针，然后把引用计数存储在 Side Table 中。这两种情况是通过在这个字段中设置一个位来区分的，这个位指示它是持有引用计数还是指向 Side Table 的指针。

Side Table 允许 Swift 保持旧的弱引用系统的基本形式，同时修复其缺陷。现在弱引用不再像以前那样指向对象，而是直接指向 Side Table。

因为 Side Table 占用内存小，所以不存在为大对象的弱引用浪费大量内存的问题。这也指出了线程安全问题的一个简单解决方案：不要预先将弱引用清零。对它的弱引用可以不用管，直到这些引用本身被覆盖或销毁。

当前的 Side Table 实现只持有引用计数和一个指向原始对象的指针。像关联对象这样的额外用途目前还只是假设性的。Swift 没有内置的关联对象功能，Objective-C 的 API 仍然使用一个全局表。

这项技术有很大的潜力，我们可能会在不久之后看到类似关联对象的使用，比如在类的 extension 里面增加存储属性。

关于最新的 Side Table 的实现，可以浏览 GitHub 上的[标准库源码](https://github.com/apple/swift)

- [RefCount](https://github.com/apple/swift/blob/main/stdlib/public/SwiftShims/RefCount.h)
- [WeakReference](https://github.com/apple/swift/blob/main/stdlib/public/runtime/WeakReference.h)
- [HeapObject](https://github.com/apple/swift/blob/main/stdlib/public/runtime/HeapObject.cpp)

## C++ 的 weak_ptr

如果你深入的去看看 C++的内存对齐，你会发现和 Swift 很像。当我们使用`std::make_shared` 初始化时，对象的下一个字段就是强引用计数和弱引用计数。但是如果直接用`shared_ptr`初始化，引用计数会在堆里面的其他地方。

C++里面，我们可以选择一个对象是否可以有弱引用。比如类`Spaceship`默认是没有任何引用计数的，但是一个`shared_ptr<Spaceship>`有两种引用计数。

我们可以从`shared_ptr<Spaceship>`手动创建一个弱引用`weak_ptr<Spaceship>`。

`weak_ptr`遵循“零开销抽象”原则，只有当我们需要的时候才会占用 16B 的空间。

普通的指针对象`Spaceship*`无法获得`weak_ptr<Spaceship>`，除非`Spaceship`继承`std::enable_shared_from_this`

## Rust 的 Weak

Rust 的 `Rc`和`Weak` 分别基于 C++的`shared_ptr`和`weak_ptr`实现，但是有一些不同的是：

- Rust 的 `Rc` 总是将引用计数挨着对象存放，但是 C++允许我们放在独立的一块内存
- 给定`&Spaceship`，我们无法获取`Weak<Spaceship>`

现实中我们用到`Rc`和`Weak`的地方比较少，有其他类似弱引用的手段。

## 随处可见的弱引用

我们看过了上面的几种弱引用实现，其实，在平常写代码时也有很多地方用到非常像弱引用的东西。

例如，一个表示文件名的字符串`myfile.txt`，我们可以用来读取文件内容，如果文件存在的话：

```vale
func main() {
  if contents = readFileAsString("myfile.txt") {
    // File exists!
    println(contents);
  } else {
    println("File doesn't exist!");
  }
}
```

或者一个整型的 id，我们可以用来在 map 里面查找`Spaceship`:

```vale
func printName(ships &HashMap<int, Spaceship>, ship_id int) {
  if ship = ships.get(ship_id) {
    println("Ship exists! {ship.name}")
  } else {
    println("Ship doesn't exist!");
  }
}
```

请注意，我们首先检查是否存在，然后使用返回结果，就跟弱引用一样。

我最喜欢的一种变相的弱引用是**generational index**，经常在 C++和 Rust 程序中使用。

## Generational Indices

我们通常将对象存储在 array 或 vector 里面，比如 `Vec<Spaceship>`，类似于 C++ 的 `std::vector<Spaceship>` 或者 Java 的 `ArrayList<Spaceship>`。当销毁一个`Spaceship`对象时，我们通常将其指向另一个`Spaceship`对象。

有时，我们会记住`Spaceship`在 vector 索引。然后，我们可能想知道`Spaceship`是否还存在，或者是否被重用了。

我们使用一个 i64 类型的整数来记录当前`Spaceship`对象的第几代，如`Vec<Spaceship, i64>`。每次我们重用对象时，就增加这个数值。

无论什么时候我们想要记住`Spaceship`的索引，我们都能与此同时记住它是第几代。

为了方便，我们可以把索引值和这个代数放在一个小 struct 里面，取名`GenerationalIndex`:

```c
struct GenerationalIndex {
  index: i64,
  remembered_generation: i64
}
struct Missile {
  enemy_ref: GenerationalIndex
}
```

现在，如果我们想知道`Spaceship`是否还存在，只须比较当前代数是否等于可记住的代数，如：

```c
if enemies[missile.enemy_ref.index] == missile.enemy_ref.remembered_generation {
  // Enemy still exists!
  let enemy = &enemies[missile.enemy_ref.index];
  ...
}
```

这就是 Generational Index,类似弱引用！

然而，Generational Index 有一个缺点：为了 "解除引用"，我们需要访问包含的 Vec（比如上面的 Vec 类型 `enemies`），通常是通过一个参数传入。

这有时会很不方便：当我们添加一个新的参数时，我们必须改变我们的调用者来提供它，然后是我们的调用者的调用者，以及我们的调用者的调用者的调用者，这可能导致 "重构冲击波"。过于频繁地这样做会导致 API 面目全非。

有时为了避免这个问题，我们会把所有的容器放到一个 "god object"中，类似于一个全局对象，并把它作为一个参数传递给我们代码库中的每个函数。

也许有一个更好的方法来解决这个缺点。请继续阅读!

## Vale 的 Generational References

在 Vale 语言里，我们加了一个类似 generational index 的东西，命名为 **Generational References**

在内存里，每个对象都有一个"当前代数"的数值紧挨着它。无论什么时候我们销毁一个对象时，我们都增加这个数值。

为了创建这个对象的弱引用，我们做了两件事：

- 一个指向这个对象的指针
- 这个对象的当前代数

然后把它们两个绑在一起。

想要知道这个对象是否仍然存在，Vale 只需要比较这个对象的当前代数和我们引用的可记住的代数。

当我们实现完 Generational References 后发现，它比引用计数快 2.3 倍。

相比其他弱引用方法，它有如下优势：

- 创建或销毁一个弱引用无需额外操作，不需要增加引用计数。
- 方便安全的 FFI，C 代码不会污染 Vale 对象。
- 内存销毁只需要 8B

不足之处：

- 需要一些虚拟内存操作来让 OS 释放内存
- 只能用于有稳定地址的堆内存

由此可见，Vale 程序混合了以下三种方法：

- 堆内存，我们使用 generational references
- 当我们能够方便的访问容器时，我们使用 generational indexes
- 其他所有情况，我们使用增强版的 generational indexes，将一个容器引用打包进来

通过在标准库里面提供这些方法，我们可以非常容易就拥有了快捷的弱引用。这就是我们的目的，让速度和安全比以前更容易。
