---
layout: post
title: Swift 中基于闭包的 KVO 的隐患
subtitle:
author: ddddxxx
tags:
    - Swift
---

KVO (Key-Value Observing) 是 Foundation 提供的极其强大的特性，但是使用起来十分啰嗦。好在 Swift 4 提供了一个简洁的包装。我们可以轻松写出以下的代码：

```swift
class A: NSObject {
    @objc dynamic var v: Int = 0
}

class B: NSObject {

    let a = A()
    var token: NSKeyValueObservation?

    override init() {
        super.init()
        token = a.observe(\.v, options: [.new]) { _, change in
            print("new value: \(change.newValue!)")
        }
    }
}
```

根据文档，我们甚至不用在销毁 `B` 时手动移除观察者，因为 NSKeyValueObservation 在生命周期结束后就会自动停止观察：

> when the returned NSKeyValueObservation is deinited or invalidated, it will stop observing.

真的是这样吗？我们来加两个符号断点：

- `-[NSObject addObserver:forKeyPath:options:context:]`
- `-[NSObject removeObserver:forKeyPath:context:]`

然后测试一下：

```swift
var b: B? = B()
b = nil
```

运行到第一行时，我们可以看到确实调用了`addObserver`，然而，`removeObserver` 却没有如预期调用。这可不是我们想要的结果，带着观察者被销毁可能会导致一些难以调试的崩溃。

如果我们在第一行和第二行之间插入 `b.token = nil`，手动释放 KVO，可以发现此时又会调用 `removeObserver`，而靠 `B` 来自动释放就会出问题。可以说是非常玄学了。

更加玄学的是，只要调换一下成员声明的顺序，整个问题就消失了：

```swift
-     let a = A()
      var token: NSKeyValueObservation?
+     let a = A()
```

有些同学可能已经猜到问题在哪了。`NSKeyValueObservation` 内保存了一个观察对象的弱引用，以便在合适的时机注销观察者。第一个例子中，`B.a` 先销毁，等到 `B.token` 销毁时，已经找不到目标来注销观察者了。

这里有个非常令人迷惑的逻辑漏洞，token只被 `self` 引用，保证了token的生命周期不长于 `self`。而 `self` 引用了观察对象，保证了观察对象生命周期不短于`self`。那么观察对象被销毁前，token也会被销毁。如此就保证了观察对象就不会带着观察者被销毁。而事实上，这还取决于成员变量销毁的顺序。

为了避免这种时机问题，最保险的办法是，在 `B.deinit` 中手动销毁 KVO。

```swift
+     deinit {
+         token?.invalidate()
+     }
```

事实上，第一个例子中的代码并不完全符合[苹果官方的示例](https://developer.apple.com/library/content/documentation/Swift/Conceptual/BuildingCocoaApps/AdoptingCocoaDesignPatterns.html)。如果按照示例来改写的话，应该是这样的：

```swift
...
-     let a = A()
+     @objc let a = A()
...
-         token = a.observe(\.v, options: [.new]) { _, change in
+         token = observe(\.a.v, options: [.new]) { _, change in
...
```

由于观察对象变成了 `self`，就不会出现上述的问题了。不过需要把对象标注为 `@objc`，我并不喜欢这种方式。

## TL;DR

如果直接对观察对象调用 `observe(_:options:changeHandler:)` 并保存 token，可能导致观察对象早于 token 被释放。需要在 `deinie` 中手动调用 token 的 `invalidate` 方法以注销观察者。

















