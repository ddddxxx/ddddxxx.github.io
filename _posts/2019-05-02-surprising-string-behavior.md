---
layout: post
title: Swift 中 String 和 Index 的奇妙行为
subtitle:
author: ddddxxx
tags:
    - Swift
---

在 Playground 中运行这段代码，猜猜最后一行的结果是什么：

```swift
var s = String(format: "你好 %@", "world!")
let i = s.firstIndex(of: "w")!
s.reserveCapacity(100)
s[i]
```

答案是 Xcode (10.2.1, 10E1001) 会崩溃。如果把最后一行换成 `print(s[i])`，你会得到 `\345`。

无论如何，这都不是我们预期的结果。在第二行，我们获取了 `w` 这个字符的索引。然后我们改变了字符串容器的大小。这显然没有改变字符串的内容，所以上一行的索引仍然有效。最后我们使用这个索引寻找对应的字符，发现 `w` 不在那了。

如果你使用的 Swift 版本低于 Swift 5（即 Xcode 低于 10.2.0），则不会有这个问题。如果把第一行换成 "你好 world!"，同样不会有问题。可能有些人已经猜到了问题所在，不过还是让我们从头说起。

# String 和 Index

要解开这个问题，我们首先要知道 `String.Index` 是怎么工作的。从外部来看，这是一个不透明结构体。而根据[源代码](https://github.com/apple/swift/blob/85a30989fa8fabc599200f9d29864d8009fb3cd7/stdlib/public/core/StringIndex.swift#L39)我们可以看到，其中只包含了一个 `UInt64 ` 的 `_rawBits`。抛开其中缓存和转码相关内容，和索引相关的是其中的高 48 位。“An offset into the string's code units”，这一偏移量是基于码元（Code Unit）的。我们可以通过 `encodedOffset` 直接访问这一值。

注意，`String.Index` 是基于码元的偏移量。而同一个码点，使用不同的编码方式，得到的码元数量也不同。所以同一个字符串的同一个位置，在不同编码下，偏移量是不同的。

这时候原因就很明显了，`reserveCapacity` 这个操作，改变了字符串的编码方式，同时作废了相关的所有 `Index`。使用无效的 `Index` 当然不能得到正确结果。

# 编码

但我们还要问，为什么编码会变？

继续看源码，[一个字符串对象可能以三种方式存储](https://github.com/apple/swift/blob/85a30989fa8fabc599200f9d29864d8009fb3cd7/stdlib/public/core/StringObject.swift#L83-L85)。忽略静态不可变字符串，一个 `String` 对象的底层还可能是原生 Swift String，或是桥接来的 `NSString`。我们已经知道 `NSString` 是用 UTF-16 编码，那 Swift String 呢？

好吧，正如你猜到的，[从 Swift 5.0 起，native Swift string 的编码方式由 UTF-16 切换到 UTF-8](https://forums.swift.org/t/string-s-abi-and-utf-8/17676)。

回到原来的问题，字符串和 index 由执行 `reserveCapacity` 之前的

```
你    好    ␣     w     o     r     l     d     !
\4F60 \597D \0020 \0077 \006F \0072 \006C \0064 \0021
                  ^ index
```

变成了

```
你          好          ␣   w   o   r   l   d   !
\E4 \BD \A0 \E5 \A5 \BD \20 \77 \6F \72 \6C \64 \21
            ^ index
```

好像哪里不对。虽然指向了不同的字符，但还在正确的字符边界上。为什么还是会崩溃呢？

# Cache

我们先尝试使用同一个偏移来制作下标：

```swift
"你好 world!"[String.Index(encodedOffset: 3)]
// 好
```

确定不是因为偏移导致的崩溃。由于 Index 是简单结构体，我们可以直接强转成其中的 `_rawBits`

```swift
String(format: "%X", unsafeBitCast(i, to: UInt64.self))
// 30100
String(format: "%X", unsafeBitCast(String.Index(encodedOffset: 3), to: UInt64.self))
// 30000
```

找到区别了，b8 开始多了个 1。我们可以直接查源码：

> b13:b8: grapheme cache: A 6-bit value remembering the distance to the next grapheme

这几位缓存了到下一个字符的距离。可以加速下标访问。回到我们的字符串，使用 UTF-16 时，这个距离是 1，使用 UTF-8 时，这个距离是3。我们可以尝试构造出这个下标

```swift
let i = unsafeBitCast(0x30300, to: String.Index.self)
"你好 world!"[i]
// 好
```

当我们使用 30100 这个下标时，只有一个码元被取出，得到 `0xE5`，即为文章开头打印出的 `0o345`。Xcode 试图使用这个非法的 `Character` 构造 `String`。导致了崩溃。

# 迷思

事实上这个问题在 Swift 5 之前就存在了，如果你在一个静态 c 字符串上计算 index，然后拷贝一份，稍作修改再索引，也可能得到未定义行为。但是 Swift String 和 `NSString` 的分裂加重了问题的影响范围。现在对一个 `String` 做出最轻微的修改，甚至不必修改其内容，就会作废之前的所有的 index （而非修改处之后的 index）。这在使用 `NSRegularExpression` 时尤其恼人。

要避免这一问题，你可以：

- 使用 `String.UTF16View.Index`
  - 缺点：某些情况下性能糟糕
- 使用 `NSString`
  - 缺点：需要不断类型转换，对性能也有影响
- 使用 `[Character].Index`
  - 缺点：需要将待处理字符串转为 `[Character]`，无法使用字符串相关方法

现有的设计并不理想。很难想象 `String.Index` 居然不能兼顾到性能，安全性和易用性的任何一方。我们需要使用冗长的语法构建出 Index，并且承担了大量的性能代价。然而这些脆弱的 Index 不能承受最轻微的修改。如果这是 Unicode Correctness 的代价，未免也太重了些。
