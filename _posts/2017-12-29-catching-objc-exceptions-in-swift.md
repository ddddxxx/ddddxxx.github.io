---
layout: post
title: 在 Swift 中捕获 Objc 抛出的异常
subtitle:
author: ddddxxx
date: 2017-12-29
tags:
    - Swift
---

# 错误还是异常

一般来说错误分为两种情况。一种是可以预见，可恢复的错误。称为错误（error）。另一种则是不能预见，通常也是不可恢复的致命错误。称为异常（exception）。例如，文件无法打开是错误，数组访问越界则是异常。

在 Objective-C 中，这两种错误分别以 `NSError` 和 `NSException` 表示。Apple 有两篇关于此的文档，[Dealing with Errors](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ProgrammingWithObjectiveC/ErrorHandling/ErrorHandling.html) 和 [Handling Exceptions](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Exceptions/Tasks/HandlingExceptions.html)。

Swift 从 2.0 开始有了内建错误处理。虽然形式上是我们熟悉的 try-catch(do-catch) 但实际上做的是错误处理，而不是异常。Objc 中原有使用 `NSError` 的代码被标记为 `throws`。而原本会抛出异常的代码则使用 `fatalError` 直接崩溃。

# 接不住的异常

我个人喜欢 Swift 的设计。它要求开发者显示处理可能的错误，并且其抛出错误的效率远高于 C++ （相比兼任错误处理的异常处理），而对于异常部分，其被设计为尽早退出，避免程序进入未定义区域。但是，这样的设计也造成了一些尴尬的情况，完全排除异常处理的 Swift 无法处理 Objc 中抛出的异常。

例如，反序列化方法 `NSKeyedUnarchiver.unarchiveObject(with:)` 在失败时会抛出 `invalidArchiveOperationException`，这个错误无法通过 Swift 的 do-catch 接住。事实上，当你试图用纯 Swift 反序列化一段来源不明的数据时，没有任何办法能避免程序崩溃。

我认为这是一个设计上的失误，这个异常明显是开发阶段可以预见的，但却没有按照错误来处理。iOS 9 / macOS 10.11 引入了另一个不会抛异常的方法 `unarchiveTopLevelObjectWithData(:) throws`。可惜对于大多数应用来说，系统版本要求太高了。

# 封装异常处理

我们可以在 Objc 做一层封装，以便在纯 Swift 中处理异常：

```objective-c
void catchException(void (NS_NOESCAPE ^ _Nonnull tryBlock)(), void(NS_NOESCAPE ^ _Nullable catchBlock)(NSException * _Nonnull e)) {
    @try {
        tryBlock();
    }
    @catch (NSException *exception) {
        if (catchBlock) {
            catchBlock(exception);
        }
    }
}
```

这个方法导入到 Swift 中是这样的：`catchException(_: () -> Void, _: ((NSException) -> Void)?)`。于是我们可以这样做：

```swift
var result: Any?
catchException({
    do {
        let data = try Data(contentsOf: url)
        let obj = NSKeyedUnarchiver.unarchiveObject(with: data)
        result = obj
    } catch {
        // error handling
    }
}, { e in
    // exception handling
})
// use `result`
```

还是很不方便，这样做有几个问题：

1. 无法在 block 中返回值到外部
2. 不保证控制流
3. 和 Swift 原生 error 嵌套时难以处理

我们再用 Swift 包一层：

```swift
extension NSException: Error {
    public class func `catch`<T>(_ block: () throws -> T) throws -> T {
        var _result: T?
        var _error: Error?
        catchException({
            do {
                _result = try block()
            } catch {
                _error = error
            }
        }, { error in
            _error = error
        })
        if let _error = _error {
            throw _error
        } else {
            return _result!
        }
    }
}
```

这里作为 `NSException` 的类方法以避免污染命名空间。注意我们让 `NSException` 遵循 `Error`，这样允许我们直接抛出把 exception 作为错误抛出。

于是我们可以这样：

```swift
let obj = try NSException.catch { () -> Any? in
    let data = try Data(contentsOf: url)
    return NSKeyedUnarchiver.unarchiveObject(with: data)
}
// use `obj`
```

错误处理则是这样：

```swift
do {
    // try NSException.catch
} catch let error as NSException {
    // exception handling
} catch {
    // error handling
}
```

无论如何，捕获 `NSException` 并允许程序继续运行是极不推荐的。Objc 被假定不能从异常中恢复，所以 ARC 默认不保证异常安全，抛出异常可能导致内存泄漏。务必只在必要时这样做。
