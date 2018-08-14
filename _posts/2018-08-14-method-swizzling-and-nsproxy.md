---
layout: post
title: Method Swizzling 与 NSProxy
subtitle:
author: ddddxxx
tags:
    - Runtime
---

> 本文假设读者有关于 Method Swizzling，和 objc 消息处理流程的基本知识。

## Hook delegate 方法

使用 Method Swizzling 可以轻松 hook 一个类中的已知方法，但是对于 delegate 方法，其实际调用的类是不确定的, 如何针对未知调用类来 hook 呢？

这是一个很经典的问题，网上已经有很成熟的方案了。这里只大概讲一下思路。

由于 delegate 的调用对象是一个遵循 delegate 协议的类的实例，我们首先需要 hook `setDelegate` 这个方法。找到实际的 delegate 对象，然后动态进行 Hook。这也分两种情况：

- delegate 对象实现了目标方法。则交换实现。
- delegate 对象未实现目标方法。则动态添加该方法。

我们是如何判断 delegate 对象是否实现了目标方法的呢？是通过 `class_getInstanceMethod` 来查找 class 中的实例方法，如果查到了，则认为该 class 实现了该方法。

停！这里有一个很隐蔽的漏洞。我们并不是要判断 **delegate 对象是否实现了某个方法**，而是要判断 **delegate 对象能否响应某条消息**。这两个命题一般是相等的，但在某些特殊情况下有着细微的差别。

## 坑

如果你用 Method Swizzling 实现了 hook UIScrollView 的 delegate 方法，你会遇到一个很诡异的的 Bug。具体来说，UIWebView 对点击事件的响应会相当奇怪。你可能无法和页面中的某些元素交互。

你可能想看一下动态 hook 的过程发生了什么。在 UIWebView 的例子中，scroll view 是 `_UIWebViewScrollView`，而设置的 delegate 是 `_UIWebViewScrollViewDelegateForwarder` 的实例。试图 hook 时，我们发现 delegate 并没有实现 `UIScrollViewDelegate` 中的方法，于是我们添加了这些方法。但是如果我们不添加这些方法，而是打上符号断点的话，会发现 `_UIWebViewScrollViewDelegateForwarder` 确实处理了这些消息。

如果你熟悉 proxy 设计模式的话，这个类名已经提示你发生什么事了。

## Proxy 与 消息转发

上面例子中的 `_UIWebViewScrollViewDelegateForwarder` 是一个 proxy 对象。它本身不处理任何消息，也没有实现 `UIScrollViewDelegate` 中的方法。当它收到一条消息时，会通过消息转发流程，将消息转发给真正的接收者。

所以，当你通过 `class_getInstanceMethod` 查询时，会发现 proxy 并没有这些实例方法。于是你动态添加了这些方法。然而，一旦 proxy 对象实现了某个方法，对应的消息就不会走转发流程，而是直接被 proxy 处理掉了。也就是说，动态添加的方法“吃掉”了本应被转发的消息。

我们可以通过 [dump 出来的头文件](https://github.com/nst/iOS-Runtime-Headers/blob/master/Frameworks/UIKit.framework/_UIWebViewScrollViewDelegateForwarder.h)看到 `_UIWebViewScrollViewDelegateForwarder` 到底做了什么。它在内部保存了一个真实的 delegate，然后通过 `forwardInvocation:` 和 `methodSignatureForSelector:` 将消息转发给这个 delegate。

## 解决方案

任何动态添加的方法都可能“吃掉”本应被转发的消息。不过我们可以选择不要 Hook 可能转发消息的类。改进后的 Method Swizzling 流程是这样的：

- 通过 `class_getInstanceMethod` 查找实例方法。
- 如果没有实例方法，检查 delegate 是否是 NSProxy 的子类，或者实现了 `forwardingTargetForSelector:`，`forwardInvocation:` 这一类的转发方法
- 若是，则说明可能是 proxy 对象，终止 hook 流程

更好的方法是通过 Proxy 对象拦截发送的消息。我会再写一篇文章描述这一方案。
