---
layout: post
title: 制作跨  平台的动态链接库
subtitle:
author: ddddxxx
tags:
    - Swift
    - Xcode
    - iOS
    - macOS
    - framework
---

> 注意：这篇文章旨在解决多平台项目中 target 过多的问题，而不会教你制作 Framework 的基础知识。

## 引言

如果你制作过跨  平台的 framework，你一定遇到过这种情况：

<img src="/img/in-post/post-cross-platform-frameworks/Alamofire-targets.png" width="228px"/>

Alamofire 支持 iOS/macOS/tvOS/watchOS，加上测试，总共有 7 个 target。

再看 SnapKit 的：

<img src="/img/in-post/post-cross-platform-frameworks/SnapKit-targets.png" width="178px">

同样是支持多平台，然而只有两个 Target。它要怎么编译到不同的平台呢？于是我们来看看它的 scheme：

<img src="/img/in-post/post-cross-platform-frameworks/SnapKit-scheme.png" width="450px">

只需要一个 scheme！其中包含了所有的目标平台。每一个都能运行单元测试并通过。

这样的好处是显而易见的。通常多平台的项目都会在不同的 target 间使用相同的编译规则。而多个 target 意味着你需要手动逐个修改编译选项，极其容易出错。我在 SwiftyJSON 的工程中就发现了[这样的错误](https://github.com/SwiftyJSON/SwiftyJSON/issues/845#issuecomment-304756920)。

多个 target 本身也导致了更多 target，一个 test target 只能运行在一个 scheme 上。所以如果你的项目中有多个 target，你还需要创建等量的 test target 以运行单元测试。

## 解决方案

其实要做到单 target 非常简单，但是有很多细节需要注意。

单 target 的核心在于，Xcode 使用 `Supported Platforms`[^SupportedPlatforms] 来标记一个 target 支持的设备，你可以手动添加需要支持的平台。我创建一个新项目作为例子：

<img src="/img/in-post/post-cross-platform-frameworks/new-project.png" width="769px"/>

这是 `Supported Platforms` 的默认设置：

<img src="/img/in-post/post-cross-platform-frameworks/supported-platforms-original.png" width="701px"/>

添加我们需要支持的平台：

<img src="/img/in-post/post-cross-platform-frameworks/supported-platforms-modified.png" width="665px"/>

此时在右上角 scheme 中选择平台，发现已经可以选择 `My Mac` 作为目标，但是 看不到 tvOS 和 watchOS 的模拟器。这是因为除了支持的平台以外，我们还要标记支持的设备。这个选项叫做 `Target Device Family`[^TargetDeviceFamily]，可选的值有 1，2，3，4。

不幸的是，Xcode并不允许我们选择全部的组合：

<img src="/img/in-post/post-cross-platform-frameworks/target-device-family-select.png" width="673px"/>

你需要手动修改工程文件。文件的位置在 `PATH/TO/YOUR/PROJECT/YourProjectName.xcodeproj/project.pbxproj`。搜索 `TARGETED_DEVICE_FAMILY`，将对应行的值改为 `1,2,3,4`。

<img src="/img/in-post/post-cross-platform-frameworks/target-device-family-modified.png" width="432px"/>

> 如果文件中没有对应字段，请在 Xcode 中修改 `Target Device Family` 的值，并确认该行变为加粗字体。（细字体意味着 pbxproj 中没有对应字段，自动取默认值）

现在你应该可以在 scheme 中选择所有的设备和模拟器了，你还可以在任一平台上编译。不过还有一些事要处理，是关于动态链接库的。

iOS 和 macOS 的程序包中，动态链接库的位置是相同的，都在 `(Contents/)Frameworks／`。然而二者的可执行文件位置却不同。iOS 的可执行文件直接位于最外层，而 macOS 的可执行文件位于 `Contents/MacOS/`。所以 iOS 中 framework 的相对位置是 `Frameworks/`，而 macOS 中则是 `../Frameworks/`。我们需要设置不同的加载地址。

这个选项叫做 `Runpath Search Path`，我们需要额外设置针对 macOS 的选项。

<img src="/img/in-post/post-cross-platform-frameworks/runpath-search-path.png" width="697px"/>

展开该项目，点击 `+` 符号以添加一个针对 macOS 的设置。

<img src="/img/in-post/post-cross-platform-frameworks/runpath-search-path-modified.png" width="682px"/>

将 macOS 的值改为 `@loader_path/../Frameworks` 和 `@executable_path/../Frameworks`。并在 `Release` 中进行相同的设置。

## 迁移现有的项目

你可能已经有包含数个 target 的 framework。你可以遵循下列步骤将其合并为单个 target：

1. 选定一个平台的 target 作为目标结果。

2. 遵照[上一节](#解决方案)的步骤使其支持多个平台。

3. 检查其他 target 的编译选项。对于在 Xcode 中显示为加粗的项目，将其选项合并到目标 target 中。

4. 你的源文件中可能有一些仅用于特定平台的代码。你需要将其添加到目标 target 中，并使用 `#if TARGET_OS_XXX`(Objc), `#if os(XXX)`(Swift) 等方法使其仅在特定平台编译。

5. 如果你为每个 target 指定了不同的 `Info.plist`，我强烈建议你只保留一份。通常来说它们的内容都是相同的。

6. 为目标 target 选择不同的平台编译并测试。通常来说这不会有任何问题。

7. 删除剩余的 target。

## 开始新的跨平台项目

你可能已经看过一些文章教你如何制作多平台 framework。通常办法是新建一个空的 Xcode 工程，然后依次添加每个平台的 target。整理文件夹以使用同一份源代码。这些都完成后，你就可以遵照[上一节](#迁移现有的项目)的步骤合并这些 target。

不过还有更好的方法。如果你使用 Swift Package Manager 的话，这一切都会简单到难以置信。

To be continued...

## 引用 Carthage 包

To be continued...

## 注

[^SupportedPlatforms]: `Supported Platforms` 可选值集合
    - macosx
    - iphoneos
    - iphonesimulator
    - appletvos
    - appletvsimulator
    - watchos
    - watchsimulator

[^TargetDeviceFamily]: `Target Device Family` 数字对应关系
    - 1 = iPhone and iPod touch
    - 2 = iPad
    - 3 = Apple TV
    - 4 = Apple Watch