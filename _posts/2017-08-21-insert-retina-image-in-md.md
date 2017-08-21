---
layout: post
title: 如何优雅地在 Jekyll 博客中插入高清图片
subtitle:
author: ddddxxx
tags:
    - Jekyll
    - CSS
---

用 Markdown 插入图片是一件很简单的事，只需要这样写：

```markdown
![AltText](/path/to/img.jpg)
```

然而，如果你需要进行更多的控制，情况立刻变得复杂了起来。由于 Markdown 秉持结构和样式分离的原则，你无法在插入图片时控制其大小。要达到这样的目的，你需要使用普通的 `<img>` 标签：

```html
<img src="/path/to/img.jpg" width="100" height="100" alt="AltText" />
```

相对前一种语法，后者明显麻烦许多。所幸这种情况一般都是极少数。一般来说，我们只需要使用前一种语法。然而，有一种状况会让你需要大量使用后一种语法 —— Retina 屏幕的截图。一张 100pt * 100pt 的截图事实上是 200px * 200px。贴到文章里之后，电脑又会以 200pt * 200pt 的大小来渲染。你的截图立刻变得大而模糊。为了避免这种状况，你需要手动计算图片的实际大小，然后使用 HTML 语法插入图片。当你需要插入大量图片时，这一工作很快变得枯燥而难以忍受。

当然，你可以通过指定比例从而避免手工计算大小：

```html
<img src="/path/to/img.jpg" width="50%" alt="AltText" />
```

虽然免去了计算大小，你依然需要使用冗长的 HTML 语法。能不能用 Markdown 语法来做这件事呢？

我们知道有些 Markdown 方言支持调整图片大小，例如：

```markdown
Mou:
![AltText](/path/to/img.jpg =100x)

MWeb:
![AltText-w100](/path/to/img.jpg)

...
```

显然，这些在 Jekyll 中都是不支持的。不过，我们有更好的方法：

```markdown
![AltText @2x]()
```

只需添加 `@2x` 后缀，图片就会自动缩放为50%大小

这显然不是 Markdown 或 Jekyll 的功能，在你的博客中使用不会有任何效果。真正的魔法在 CSS 样式表中：

```css
.post-container img[alt~="@2x"] {
    zoom: 50%;
}
```

我们用到了 CSS 中的属性选择器，它可以对拥有指定属性的 HTML 元素设置样式。在这里，我们对 alt 中包含 `@2x` 的所有图片添加了 `zoom: 50%` 属性。

同理，你也可以为 iPhone * Plus 的截图设置三倍缩放：

```css
.post-container img[alt~="@3x"] {
    zoom: 33%;
}
```
