---
layout: post
title: 封装 NSRegularExpression
subtitle:
author: ddddxxx
tags:
    - Swift
---

> 本文是关于 [Regex](https://github.com/ddddxxx/Regex) 库的设计心得。

在 Swift 中使用 `NSRegularExpression` 非常繁琐。实际使用中，我们通常都会对其进行扩展（例如去除匹配时必须传入的 `NSRange` 参数），或是另起炉灶，重新设计一套 API。

出于个人使用需求，我早在 Swift 3 时期就制作了一个 [Regex](https://github.com/ddddxxx/Regex) 库。当时我找遍了 GitHub 上类似的项目，无一能满足我的要求，只好自己造轮子。如今两年过去了，这样的项目在数量上多了不少，但逐一看来，质量竟还不如我两年前的设计。遂决定翻修项目，增加文档说明，发布 1.0.0 版本，并在此记录一些设计思路。

# 整体设计

从正则表达式本身的属性出发，很容易得出基本的设计：

1. 每个正则表达式匹配结果都是一个 `Match` 实例
2. 可以直接从 `Match` 中获取范围和匹配的字符串，不需要像 `NSTextCheckingResult` 一样再截取一次
3. 可以从 `Match` 中取得 capture group，并从 capture group 中获取范围和匹配的字符串
4. 可以获取到 0 号 capture group，即整个匹配

从第 4 点出发，我们很容易得出 `Match` 就是 capture group 的集合。`Match` 本身不必携带额外的信息，只需从 `$0` 中获取即可。那么 capture group 就不能是简单的 String, 内部要有自己的结构才行，即 `Capture` 的实例。

此时 `Match` 的设计大概是这样的：

```swift
struct Match {
    let captures: [Capture?]
}
```

有两个要注意的点

1. `Match` 和 `Capture` 必须是 struct 而不是 class。二者的实例可能非常小且数量多。值类型可以避免申请内存和引用计数的开销
2. `Capture` 可空，因为正则表达式允许 capture group 不参与匹配

`Capture` 的内部是什么样的呢？显然它应该包含范围和字符串。为了避免不必要的字符串复制，这里应该使用 `Substring` 而非 `String`。由于 `Substring` 自带范围，我们不需要额外储存范围：

```swift
struct Capture {
    let content: Substring
    var range: Range<String.Index> {
        return content.startIndex..<content.endIndex
    }
}
```

看起来不错。但是不行，我们踩进了第一个坑里。

# Unicode

这是正则表达式和 Swift 间不可调和的矛盾。正则表达式（ICU标准，即 `NSRegularExpression` 采用的标准）匹配的单位是 Unicode Scalar，可以大致看成码点(Code Point)，而 Swift 中字符（Character）的标准是 Extended Grapheme Cluster。所以 Swift 中一个字符可能被拆成多个码点分别匹配。例如 `é`(`U+0065 U+0301`) 在 Swift 中是一个字符，但可以从中单独匹配出一个`e`。匹配到的范围无法用 `Range<String.Index>` 表达。

这并非是罕见情况，`CRLF`就是一个组合字符（`"\r\n".count == 1`），Emoji 中也有大量组合字符。GitHub 上大量项目遇到不可表达的范围时就返回空字符串，这是不可容忍的。

但完全退回 Foundation的世界，只使用 `NSString` 和 `NSRange` 也不可取。`NSString` 毕竟是二等公民，比不上精心调优的 `String`，特别是对于小字符串，`String` 值类型的优势是 `NSString` 无论如何都赶不上的。

我的做法是对于有效的范围使用 Substring，无效的范围则回退到 NSString。在保证正确的前提确保性能。

# 桥接

由于 Objective-C 头文件的导入规则，`NSRegularExpression` 相关的函数签名使用的是 Swift 原生字符串。这给了很多人一个幻觉，认为 `NSRegularExpression` 可以直接操作 `String`。但事实上这个字符串是被桥接过去的。虽然桥接没有开销，但由于底层编码不同，对字符串的操作需要支付转换编码的代价。最糟糕的是，你可能需要反复支付这个代价。GitHub 上一些类似的项目在不经意间造成了 O(n^2) 的额外开销。

事实上，在匹配前转换成 `NSString` 并一次性支付转换开销对性能更有帮助。在测试中，仅这一项就导致了10倍的性能差异。即使你使用裸 `NSRegularExpression`，也要小心这一点。

# 构造器

这是一个语法糖，我没见过其它项目用过，但是我认为它很甜。

`Regex` 的构造器显然是可以抛出错误的。但是如果我们使用编译期已知的常量字符串，这样的错误就没必要处理。这是程序编写有误，直接崩溃就好了。利用 `StaticString`，我们可以重载一个不会抛出错误的构造器。除此之外，这里还需要一个非公开的标签 `@_disfavoredOverload` 来调整优先级

```swift
init(_ staticPattern: StaticString, options: Options = [])
@_disfavoredOverload
init(_ pattern: String, options: Options = []) throws
```

这样用常量字符串来初始化的时候就不需要处理错误了：

```swift
let regex = Regex("(foo|bar)") // 不会抛出错误，出错直接崩溃

let pattern = "(foo|bar)"
let regex = try! Regex(pattern) // 强制要求错误处理
```

# 跨平台

不知为何，[Linux 平台下 NSRegularExpression.enumerateMatches 要求传入逃逸闭包](https://github.com/apple/swift-corelibs-foundation/blob/main/Sources/Foundation/NSRegularExpression.swift#L170)，这和 macOS 下的函数签名不一致。如果要支持 Linux，这里必须用 `withoutActuallyEscaping` 转换一下。

# 其它

还有一些小技巧。例如模式匹配运算符 `~=`，实现之后可以用 `switch` 来匹配正则。还有`ReferenceConvertible`，实现之后 `Regex` 和 `NSRegularExpression` 可以用 `as` 来互相转换。查找替换的函数叫 `replacingMatches(of:with:options:range)`，命名参考了 `String.replacingOccurrences(of:with:)`。

其实整个 `NSRegularExpression` 的 API 非常小，全部封装一遍也就一两百行代码，工作量很少。但是由于涉及到 Objective-C 和 Swift 的桥接，以及正则表达式和 Unicode 本身的复杂性，封装过程有很多小坑。事实上，如果你直接使用 `NSRegularExpression`，很容易踩进这些坑里。而好的封装有助于你避开这些坑。
