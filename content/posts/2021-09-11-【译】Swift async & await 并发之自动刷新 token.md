---
title: 【译】Swift async & await 并发之自动刷新 token
slug: building-a-token-refresh-flow-with-async-await-and-swift-concurrency
date: 2021-09-11
categories:
  - iOS
tags:
  - Swift
  - async
toc: false
---

> 原文：[Building a token refresh flow with async/await and Swift Concurrency](https://www.donnywals.com/building-a-token-refresh-flow-with-async-await-and-swift-concurrency)

Swift 5.5 引入了 async/await，更进一步简化异步代码的语法。

本文作者使用网络请求中自动刷新 token 的功能来说明用法，这是 OAuth 2 里面必备的一个功能了。

## 工作流

先来复习一下网络请求的流程，我们假设有一个`AuthManager`对象来管理 token 操作。创建一个网络请求时，从`AuthManager`对象获取一个 token。逻辑如下：

- 如果 token 为空或无效，那就跳转到登录界面；
- 如果能够获取有效 token，那就正常执行请求；
  - 如果请求成功，直接返回请求结果；
  - 如果请求失败
    - 失败原因是 token 无效
      - 需要先刷新 token，
        - 刷新失败，抛出异常
        - 刷新成功，执行请求
          - 请求成功，返回请求结果
          - 请求失败，抛出异常（即使是 token 无效也不再重试了，不然无穷无尽）
    - 失败原因是其他，直接抛出异常

<img src="https://cdn.zhangwen.site/uPic/request-flow.png" style="zoom: 67%;" />

`AuthManager`的逻辑是先检查本地 token 是否存在，如果没有，抛出异常；如果存在，检查 token 有效性；如果 token 无效，尝试刷新 token 请求；刷新成功，返回新 token；刷新失败，抛出异常，弹出登录界面。如下图所示：

<img src="https://cdn.zhangwen.site/uPic/refresh-flow.png" style="zoom: 67%;" />

接下来，我们来实现一个`AuthManager`对象，然后演示一下在`Network`对象怎么使用它。

## 实现

首先确保一件事：在任何时间内，只能有一个刷新 token 请求在执行。

这很重要，因为有很大可能的情况是多个请求在并发执行，比如首页就很多接口在调用不同的业务，每个接口都会向`AuthManager`调用`validToken()`。

所以，我们将`AuthManager`设为`actor`。Actors 是引用类型，跟 class 的区别是只允许一个任务访问他们的可变状态，这就使得同一个 actor 在多任务代码中也可安全访问，调用时使用`await`，具体使用看[官方文档 Concurrency 部分](https://docs.swift.org/swift-book/LanguageGuide/Concurrency.html)。

初步实现如下：

```swift
actor AuthManager {
    private var currentToken: Token?
    private var refreshTask: Task<Token, Error>?

    func validToken() async throws -> Token {

    }

    func refreshToken() async throws -> Token {

    }
}
```

使用实例变量来存储 token 是方便而已，真实环境下使用**Keychain**保存。

然后定义一个`Error`错误类型：

```swift
enum AuthError: Error {
    case missingToken
}
```

接下来`validToken()`就可以这样简单实现：

```swift
func validToken() async throws -> Token {
    if let handle = refreshTask {
        return try await handle.value
    }

    guard let token = currentToken else {
        throw AuthError.missingToken
    }

    if token.isValid {
        return token
    }

    return try await refreshToken()
}
```

上面的逻辑顺序是这样的：

1. 如果正在刷新 token，等待结果，确保我们返回刷新后的 token
2. 如果不在刷新中并且本地不存在 token，用户需要登录
3. 如果本地 token 存在，我们假设 token 有效，因为我们还没到过期时间
4. 如果以上情况都不存在，我们就需要刷新 token

下面轮到实现`refreshToken()`了，分两步执行，首先处理已经有并发任务正在执行刷新操作的情况：

```swift
func refreshToken() async throws -> Token {
    // 第一步
    if let refreshTask = refreshTask {
        return try await refreshTask.value
    }

    // 第二步
    let task = Task { () throws -> Token in
        defer { refreshTask = nil }

        // 通常在这里调用网络请求，比如这样：
        // return await networking.refreshToken(withRefreshToken: token.refreshToken)

        // 这里只是做个假的id，用UUID()生成了
        let tokenExpiresAt = Date().addingTimeInterval(10)
        let newToken = Token(validUntil: tokenExpiresAt, id: UUID())
        currentToken = newToken

        return newToken
    }

    self.refreshTask = task

    return try await task.value
}
```

因为`AuthManager`是 actor 了，所以第一步就很简单了，不需要同步队列或者加锁来确保并发调用`refreshToken()`了。

第二步就是初始化一个刷新 token 的`Task`，并保存起来，然后直接返回刷新任务的结果。创建`Task`的时候，先用`defer`来确保在完成任务之前能将`refreshTask`设为`nil`。我们不需要`await`等待`refreshTask`，因为这个`Task`会自动在`AuthManager` actor 上运行。

## 在 Networking 中使用 AuthManager

代码如下：

```swift
class Networking {

    let authManager: AuthManager

    init(authManager: AuthManager) {
        self.authManager = authManager
    }

    // 加载请求
    func loadAuthorized<T: Decodable>(_ url: URL) async throws -> T {
        // 这里有可能获取token失败，所以需要try await
        let request = try await authorizedRequest(from: url)
        let (data, _) = try await URLSession.shared.data(for: request)

        let decoder = JSONDecoder()
        let response = try decoder.decode(T.self, from: data)

        return response
    }

    // 所有请求都包一层token
    private func authorizedRequest(from url: URL) async throws -> URLRequest {
        var urlRequest = URLRequest(url: url)
        let token = try await authManager.validToken()
        urlRequest.setValue("Bearer \(token.value)", forHTTPHeaderField: "Authorization")
        return urlRequest
    }
}
```

在这里我们对比一下传统的基于回调的方法，或者 RxSwift 和 Combine 的响应式方法，就可以看出**async / await**是多么直观。

到这一步，我们已经实现了下图中的绿色部分流程：

<img src="https://cdn.zhangwen.site/uPic/request-flow-progress.png" style="zoom:67%;" />

接下来的几个步骤，我们需要做点小小的改进：允许重试一次。还有，我们还需要判断一下请求结果是否`401`未认证状态，如果是，那我们需要刷新 token，然后尝试重新发起请求。这种情况不是很常见，但因为用户的设备时间可能会改动，比如换个时区，这就与服务器时间不一致了，本地不过期，但实际在服务器端是过期的。

`loadAuthorized` 改成下面的逻辑判断：

```swift
func loadAuthorized<T: Decodable>(_ url: URL, allowRetry: Bool = true) async throws -> T {
    let request = try await authorizedRequest(from: url)
    let (data, urlResponse) = try await URLSession.shared.data(for: request)

    // 检查http状态码，如果是401，则刷新token，重试请求。
    if let httpResponse = urlResponse as? HTTPURLResponse, httpResponse.statusCode == 401 {
        if allowRetry {
            _ = try await authManager.refreshToken()
            return try await loadAuthorized(url, allowRetry: false)
        }

        throw AuthError.invalidToken
    }

    let decoder = JSONDecoder()
    let response = try decoder.decode(T.self, from: data)

    return response
}
```

当然，上面的处理逻辑有点简单粗暴。你还可以处理其他非`200`状态的 http 响应，然后转成`Error`处理。

## 总结

在本文中可以看到，Swift 5.5 增加的并发部分功能是大大减少了写异步代码的繁琐程度，通过`async/await`可以将异步代码同步化，更符合程序员的思考流程。这个语法糖，其实在 Python、ES6+、Dart 等语言上早都支持了，Swift 4 的时候就有人提过 issue 了，但一直到现在才实现这个功能，算是迟来的幸福吧。

PS：如果要测试`async/await`功能，需要安装 Xcode 13 版本才可以。（截止 2021-08-24， Xcode 13 版本还是 beta 5）

## 参考

1. [Concurrency — The Swift Programming Language (Swift 5.5)](https://docs.swift.org/swift-book/LanguageGuide/Concurrency.html)
