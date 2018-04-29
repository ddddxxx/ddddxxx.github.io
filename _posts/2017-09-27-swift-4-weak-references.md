---
layout: post
title: 【译】Swift 4 中的弱引用
subtitle: Swift 4 Weak References
author: ddddxxx
date: 2017-09-27
tags:
    - 翻译
    - Swift
---

> 本文译自 [Friday Q&A 2017-09-22: Swift 4 Weak References](https://mikeash.com/pyblog/friday-qa-2017-09-22-swift-4-weak-references.html)，作者 [Mike Ash](https://mikeash.com/)。
> 
> 译者 [ddddxxx](https://ddddxxx.github.io/)，自由转载，请注明出处。

Swift 刚开源不久，我写了一篇关于[弱引用如何实现的文章](https://mikeash.com/pyblog/friday-qa-2015-12-11-swift-weak-references.html)。时过境迁，这一实现已经发生了改变。我将谈谈当前的实现方法，以及和旧版本的对比。

<!--Soon after Swift was initially open sourced, I wrote an article about how weak references are implemented. Time moves on and things change, and the implementation is different from what it once was. Today I'm going to talk about the current implementation and how it works compared to the old one, a topic suggested by Guillaume Lessard.-->

## 原有的实现

<!--Old Implementation-->

鉴于有些人已经忘了原有的实现，且不想读之前的文章，让我们快速回顾一下它是怎么实现的

在原来的实现中，Swift 对象有两个引用计数：强引用计数和弱引用计数。当强引用计数归零，而弱引用计数仍不为零时，这个对象将被销毁（destroy），但内存不会被释放（deallocate）。这导致被弱引用指向的对象成为某种僵尸对象驻留在内存中。

加载弱引用时，runtime 将会检查对象是否为僵尸对象。若是，则清零弱引用，并将弱引用计数减 1。当弱引用计数归零时则释放内存。这意味着一旦访问了对它们的弱引用，僵尸对象就会被清除。

我喜欢这个简洁的实现，但是它也有缺陷。一个缺陷是，僵尸对象可能在内存中停留很长时间。对于那些有大型实例的类（因为它包含很多成员变量，或使用 `ManagedBuffer` 这样的方法来分配额外内存），可能是极大的浪费。

编写之前的文章后我[还发现了另一个问题](https://bugs.swift.org/browse/SR-192)，它在并发访问时不是线程安全的。这个问题随后修复了，但是对其的讨论表明，它的实现者需要更好的方法来实现弱引用，为了能更好的适应类似的状况。

<!--For those of you who have forgotten the old implementation and don't feel like reading through the last article, let's briefly recall how it works.

In the old implementation, Swift objects have two reference counts: a strong count and a weak count. When the strong count reaches zero while the weak count is still non-zero, the object is destroyed but its memory is not deallocated. This leaves a sort of zombie object sitting in memory, which the remaining weak references point to.

When a weak reference is loaded, the runtime checks to see if the object is a zombie. If it is, it zeroes out the weak reference and decrements the weak reference count. Once the weak count reaches zero, the object's memory is deallocated. This means that zombie objects are eventually cleared out once all weak references to them are accessed.

I loved the simplicity of this implementation, but it had some flaws. One flaw was that the zombie objects could stay in memory for a long time. For classes with large instances (because they contain a lot of properties, or use something like ManagedBuffer to allocate extra memory inline), this could be a serious waste.

Another problem, which I discovered after writing the old article, was that the implementation wasn't thread-safe for concurrent reads. Oops! This was patched, but the discussion around it revealed that the implementers wanted a better implementation of weak references anyway, which would be more resilient to such things.-->

## 对象数据

<!--Object Data-->

Swift 中的“一个对象”是由多个部分构成的。

首先，最明显的是，源代码中声明的的存储属性。它们可以由程序员直接访问。

第二部分，是对象的类。它被用于动态派发，以及内建的 `type(of:)` 函数。大多数情况下，它是隐藏的，尽管动态派发和 `type(of:)` 暗示了它的存在。

第三部分，是各种引用计数。它们完全不可见，除非你做一些骚操作，例如读对象的原始内存，或者说服编译器让你调用 `CFGetRetainCount`。

第四部分，通过 Objective-C runtime 添加的辅助信息，例如 Objective-C 弱引用表（Objective-C 会单独追踪每个弱指针）和关联对象（associated object）。

你要把这些信息储存在哪？

<!--There are many pieces of data which make up "an object" in Swift.

First, and most obviously, there are all of the stored properties declared in the source code. These are directly accessible by the programmer.

Second, there is the object's class. This is used for dynamic dispatch and the type(of:) built-in function. This is mostly hidden, although dynamic dispatch and type(of:) imply its existence.

Third, there are the various reference counts. These are completely hidden unless you do naughty things like read the raw memory of your object or convince the compiler to let you call CFGetRetainCount.

Fourth, you have auxiliary information stored by the Objective-C runtime, like the list of Objective-C weak references (the Objective-C implementation of weak references tracks each weak reference individually) and associated objects.

Where do you store all of this stuff?-->

在 Objective-C 中，类型信息和存储属性（即实例变量）都内联存储在对象的内存中。类型占用了第一个指针大小的块，实例变量尾随其后。辅助信息存储在额外的表中。当你使用关联对象（associated object）时，runtime 会查找一个大哈希表，它的键是对象的指针。这有些慢，并且这一操作必须加锁以避免多线程操作失败。引用计数有时保存在对象的内存中，有时保存在外部表中，这取决于系统版本和 CPU 架构。

在 Swift 原有的实现中，类，引用计数和存储属性都内联存储。而辅助信息仍然存储在单独的表中。

让我们把这些语言的实现放在一边，先问一个问题：它们*应该*怎样存储？

每个位置各有优劣。存储在对象的内存中的数据访问起来很快，但始终占用空间。存储在额外表中的数据访问起来稍慢，但是若是对象没有这些数据，它们就不占用空间。

Objective-C 传统上不在对象内部保存引用计数，至少有一部分原因就是为此。Objective-C 加入引用计数时，电脑性能远不如现在，并且内存极为有限。典型的 Objective-C 程序中的大多数对象只有一个所有者，即引用计数为 1。在对象内存中留出 4 字节的空间用于存储 1 是很浪费的。通过使用外部表，1 这样的常见值可以由表项不存在来表示，从而减少内存消耗。

<!--In Objective-C, the class and stored properties (i.e. instance variables) are stored inline in the object's memory. The class takes up the first pointer-sized chunk, and the instance variables come after. Auxiliary information is stored in external tables. When you manipulate an associated object, the runtime looks it up in a big hash table which is keyed by the object's address. This is somewhat slow and requires locking so that multithreaded access doesn't fail. The reference count is sometimes stored in the object's memory and sometimes stored in an external table, depending on which OS version you're running and which CPU architecture.

In Swift's old implementation, the class, reference counts, and stored properties were all stored inline. Auxiliary information was still stored in a separate table.

Putting aside how these languages actually do it, let's ask the question: how should they do it?

Each location has tradeoffs. Data stored in the object's memory is fast to access but always takes up space. Data stored in an external table is slower to access but takes up zero space for objects which don't need it.

This is at least part of why Objective-C traditionally didn't store the reference count in the object itself. Objective-C reference counting was created when computers were much less capable than they were now, and memory was extremely limited. Most objects in a typical Objective-C program have a single owner, and thus a reference count of 1. Reserving four bytes of the object's memory to store 1 all the time would be wasteful. By using an external table, the common value of 1 could be represented by the absence of an entry, reducing memory usage.-->

每一个对象都有一个类，并且会经常访问。每次调用动态方法都需要它。它应该直接保存在对象内存中。存储在外部也不会有任何好处。

存储属性预计可以快速访问，对象是否拥有它们是在编译期决定的。如果一个对象没有没有存储属性，就可以不分配空间。所以它们应该直接保存在对象内部。

每一个对象都有引用计数。虽然不是所有对象的引用计数都不为 1，但这仍然是很常见的情况，而且现在内存已经很大了。它或许应该直接保存在对象内存里。

大多数对象都没有弱引用计数或是关联对象。为它们在对象内存中分配空间是很浪费的。它们应该保存在外部。

这是合理的权衡，但也很讨厌。对于那些有弱引用计数和关联对象的对象来说，这实在有些慢。我们怎样能解决这个问题呢？

<!--Every object has a class, and it is constantly accessed. Every dynamic method call needs it. This should go directly in the object's memory. There's no savings from storing it externally.

Stored properties are expected to be fast. Whether an object has them is determined at compile time. Objects with no stored properties can allocate zero space for them even when stored in the object's memory, so they should go there.

Every object has reference counts. Not every object has reference counts that aren't 1, but it's still pretty common, and mem ory is a lot bigger these days. This should probably go in the object's memory.

Most objects don't have any weak references or associated objects. Dedicating space within the object's memory for these would be wasteful. These should be stored externally.

This is the right tradeoff, but it's annoying. For objects that have weak references and associated objects, they're pretty slow. How can we fix this?-->

## Side Tables

Swift 新的弱引用的实现引入了一个名为 side table 的概念。

Side table 是一个单独的内存块，用于保存一个对象的额外信息。它是*可选*的，这意味着一个对象可能有 side tables，也可能没有。那些需要 side table 功能的对象会造成额外开销，而那些不需要的对象则不必付出任何开销。

每个对象都有一个指向其 side table 的指针，而 side table 中也有一个指向原始对象的指针。side table 还可以存储其它信息，例如关联对象。

为了避免 side table 带来的 8 字节空间开销，Swift做了一个漂亮的优化。对象的第一个字是它的类，下一个字是引用计数。当一个对象需要 side table 时，第二个字将被重新用作 side table 指针。因为对象仍需引用计数，所以引用计数将保存在 side table 中。这两种情况将由其中的一个位来区分，从而决定保存的是引用计数还是 side table。

side table 允许 Swift 保持原有引用计数的基本形式，并修复其缺陷。弱引用不再指向对象本身，而是直接指向 side table。

因为 side table 已知很小，这样就不会有弱引用指向大对象导致的内存浪费，所以问题自然就消失了。这也指明了线程安全问题的简单解决方案：不用提前清零弱引用。既然已知 side table 比较小，指向它的弱引用可以持续保留，直到这些引用自身被覆盖或销毁。

我要提醒一下，当前的 side table 实现只保存引用计数和指向原始对象的指针。其它用途，例如关联对象，只是一个假设。Swift 没有内建关联对象功能，而 Objective-C API 仍在使用全局表。

该技术有很大的潜力，我们或许能在不久的将来看到其它用法，例如关联对象。我希望这将打开在类扩展中声明存储属性等其它漂亮功能的大门。

<!--Swift's new implementation of weak references brings with it the concept of side tables.

A side table is a separate chunk of memory which stores extra information about an object. It's optional, meaning that an object may have a side table, or it may not. Objects which need the functionality of a side table can incur the extra cost, and objects which don't need it don't pay for it.

Each object has a pointer to its side table, and the side table has a pointer back to the object. The side table can then store other information, like associated object data.

To avoid reserving eight bytes for the side table, Swift makes a nifty optimization. Initially, the first word of an object is the class, and the next word stores the reference counts. When an object needs a side table, that second word is repurposed to be a side table pointer instead. Since the object still needs reference counts, the reference counts are stored in the side table. The two cases are distinguished by setting a bit in this field that indicates whether it holds reference counts or a pointer to the side table.

The side table allows Swift to maintain the basic form of the old weak reference system while fixing its flaws. Instead of pointing to the object, as it used to work, weak references now point directly at the side table.

Because the side table is known to be small, there's no issue of wasting a lot of memory for weak references to large objects, so that problem goes away. This also points to a simple solution for the thread safety problem: don't preemptively zero out weak references. Since the side table is known to be small, weak references to it can be left alone until those references themselves are overwritten or destroyed.

I should note that the current side table implementation only holds reference counts and a pointer to the original object. Additional uses like associated objects are currently hypothetical. Swift has no built-in associated object functionality, and the Objective-C API still uses a global table.

The technique has a lot of potential, and we'll probably see something like associated objects using it before too long. I'm hopeful that this will open the door to stored properties in extensions class types and other nifty features.-->

## 代码

由于 Swift 已经开源，关于这些的代码都能直接访问。

大多数关于 side table 的代码都在 [stdlib/public/SwiftShims/RefCount.h](https://github.com/apple/swift/blob/c262440e70896299118a0a050c8a834e1270b606/stdlib/public/SwiftShims/RefCount.h)

更高层次的弱引用 API，以及丰富的注释，都在 [swift/stdlib/public/runtime/WeakReference.h](https://github.com/apple/swift/blob/c262440e70896299118a0a050c8a834e1270b606/stdlib/public/runtime/WeakReference.h)

更多关于堆对象的实现和注释在 [stdlib/public/runtime/HeapObject.cpp](https://github.com/apple/swift/blob/c262440e70896299118a0a050c8a834e1270b606/stdlib/public/runtime/HeapObject.cpp)

我链接到了这些文件的特定版本，以便以后的读者也能看到我在说什么。如果你想看最新，最好的代码，请在点击链接后切换到 `master` 分支，或任何你感兴趣的内容。

<!--Since Swift is open source, all of the code for this stuff is accessible.

Most of the side table stuff can be found in stdlib/public/SwiftShims/RefCount.h.

The high-level weak reference API, along with juicy comments about the system, can be found in swift/stdlib/public/runtime/WeakReference.h.

Some more implementation and comments about how heap-allocated objects work can be found in stdlib/public/runtime/HeapObject.cpp.

I've linked to specific commits of these files, so that people reading from the far future can still see what I'm talking about. If you want to see the latest and greatest, be sure to switch over to the master branch, or whatever is relevant to your interests, after you click the links.-->

## 结论

弱引用是一个重要的语言特性。Swift 的初始实现非常聪明，有很多优势，但也有一些问题。通过添加 side table，Swift 的工程师能够在解决问题的同时保留这些优势。side table 也为未来的新功能铺平了道路。

<!--Weak references are an important language feature. Swift's original implementation was wonderfully clever and had some nice properties, but also had some problems. By adding an optional side table, Swift's engineers were able to solve those problems while keeping the nice, clever properties of the original. The side table implementation also opens up a lot of possibilities for great new features in the future.

That's it for today. Come back again for more crazy programming-related ghost stories. Until then, if you have a topic you'd like to see covered here, please send it in!-->
