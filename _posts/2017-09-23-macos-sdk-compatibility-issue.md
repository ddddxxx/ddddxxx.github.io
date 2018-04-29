---
layout: post
title: macOS 10.13 SDK 的一个错误，以及解决方案
subtitle:
author: ddddxxx
date: 2017-09-23
tags:
    - Swift
    - Cocoa
---

最近升级了 Xcode 9 GM，并把几个主要项目迁移到了 Swift 4。过程中发现了一个小坑，记下来以供参考。

## 0

Xcode 9 附带的 macOS 10.13 SDK 中有这样一个改动：

```swift
- func validModesForFontPanel(_ fontPanel: NSFontPanel) -> Int
+ func validModesForFontPanel(_ fontPanel: NSFontPanel) -> NSFontPanel.ModeMask
```

这是一个很 Swift 的改动，其中 `NSFontPanel.ModeMask` 是一个 `OptionSet`，这样我们就可以使用数组字面量来生成一个选项集合：

```swift
- return Int(NSFontPanelSizeModeMask | NSFontPanelCollectionModeMask | NSFontPanelFaceModeMask)
+ return [.size, .collection, .face]
```

看起来似乎很美好，然而我们看一下它的定义:

```swift
@available(OSX 10.13, *)
public struct ModeMask : OptionSet
```

这就很尴尬了，这个类要求 macOS 10.13 或更高的系统版本。我们显然不能接受这个条件。

## 1

看来我们只能使用旧版 API 了。来看一下原有的代码能否工作：

```swift
override func validModesForFontPanel(_ fontPanel: NSFontPanel) -> Int
// error: method does not override any method from its superclass
```

因为方法签名改了，原有的方法自然没有覆盖新的 API。所以我们删掉 `override`。

```swift
func validModesForFontPanel(_ fontPanel: NSFontPanel) -> Int
```

再次运行程序，发现 FontPanel 并没有遵循我们的设置。这是因为 Swift 4 新的特性，根据 [SE-0160](https://github.com/apple/swift-evolution/blob/master/proposals/0160-objc-inference.md)，NSObject 子类的方法不会自动生成 Objective-C 入口。所以这个方法在 Objective-C 运行时是不可见的。

根据要求，我们添加 `@objc` 标签：

```swift
@objc func validModesForFontPanel(_ fontPanel: NSFontPanel) -> Int
// error: Overriding method with selector 'validModesForFontPanel:' has incompatible type '(NSFontPanel) -> Int'
```

方法签名不一致，编译器拒绝生成 Objective-C 入口。

既然有版本限制，给整个方法加上版本限制如何？

```swift
@available(OSX 10.13, *)
override func validModesForFontPanel(_ fontPanel: NSFontPanel) -> NSFontPanel.ModeMask
// error: Overriding 'validModesForFontPanel' must be as available as declaration it overrides
```

到这里似乎所有的路都堵死了，难道只能退回到 Objective-C 来实现吗？

## 2

其实纯 Swift 的方法也是有的，不过有些复杂。有请 Method Swizzling：

```swift
class MyClass: NSObject {

    static let swizzler: () = {
        let cls = MyClass.self
        let sel = #selector(NSObject.validModesForFontPanel)
        let dummySel = #selector(MyClass.dummyValidModesForFontPanel)
        guard let dummyIMP = class_getMethodImplementation(cls, dummySel),
            let dummyImpl = class_getInstanceMethod(cls, dummySel),
            let typeEncoding = method_getTypeEncoding(dummyImpl) else {
                fatalError("failed to replace method \(sel) from \(cls)")
        }
        class_replaceMethod(cls, sel, dummyIMP, typeEncoding)
    }()

    @objc func dummyValidModesForFontPanel(_ fontPanel: NSFontPanel) -> UInt32 {
        // Do what you want
        return NSFontPanelStandardModesMask
    }

}
```

这里我们动态修改了 `NSObject.validModesForFontPanel` 的实现。为了使其生效，我们需要在合适的时机调用 `_ = MyClass.swizzler`。由于不需要原来的实现，这里使用了 `class_replaceMethod` 而不是 `method_exchangeImplementations`，事实上，这里的 MyClass 原本并没有 `validModesForFontPanel` 的实现。

这里的 swizzler 是一个小技巧，由于 Swift 3 废弃了 `dispatch_once`，这样写可以保证 swizzler 仅构造一次，即 Method Swizzling 仅执行一次。我会再写一篇文章详细解释这一方法。
