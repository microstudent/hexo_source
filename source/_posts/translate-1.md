---
title: 翻译：在Android上正确地使用style来配置你的View（而不陷入抓狂状态）(未完成！)
date: 2016-05-29 21:30:48
tags:
---

*refer:http://blog.danlew.net/2014/11/19/styles-on-android/*

我们总是很难在Android上正确的使用Style。始终会有一种潜在的挫败感。代码的结构很容易因此变得一团糟糕。你已经有多少次想要改变某些View的Style时却担心无意中破坏了某些东西？

<!-- more -->

在Android平台上开发了多年之后，我有一些关于如何正确使用style的强烈建议，关于如何使用它们而不让自己抓狂，还有一系列让我保持淡定的细则。

这篇文章讲的是关于**View**的style，所以请记住，即使Theme使用和Style类似的结构，它们也是不同的东西。或许我有一天会写一片关于**Theme**的文章，但是我还没说服我自己如何使用**Theme**而不抓狂。

抓稳扶手，老司机要发车了。这是一片长文章。

## 何时使用Styles
第一个我们必须思考的问题是，我们应该在什么时候通过写style而不是直接在View上写属性来自定义一个**View**？

**规则1：当有成倍的View使用完全相同的属性时使用Style**

这条规则能被以下例子完美的体现出来：

- 你在写一个计算器，每个按钮都应该看起来是一样的，所以创建一个名为**CalculatorButton**的Style是有意义的。
- 你有两个有着不同格式文本的页面，例如header，子header，和text。你可以通过创建**Header**，**SubHeader**，**Text**来统一他们的外观。
- 你在你的APP上有着很多缩略图，你想要让他们看起来都都一样，所以应该创建**Thumbnail**这个Style。

这些例子的共同点在于，这些**View**都不只是相同的属性——**他们在app中扮演着相同的角色**。现在，当你想要调整这些**View**的外观/造型，你可以通过编辑style来一次性改变他们。这节约了你的时间和精力，并且让这些**View**保持了一致。

---

想要节约更多的工作？使用资源引用(resource references)吧！

**规则2：在style里适当使用references**

你可以这样来定义一个sytle：

```
<style name="MyButton">
    <item name="android:minWidth">88dp</item>
    <item name="android:minHeight">48dp</item>
</style>
```

如果你希望**minWidth**根据不同的屏幕尺寸而变化时应该怎么做？你可以在每个尺寸的style文件里重复这个style(例如**sw600dp**和**sw900dp**)，但是这样你不得不同样重复**minHeight**这个属性。要是你想要同时改变这两个属性呢？一瞬间你将获得一大堆到处定义好了的**MyButtons**，而且每一个都是重复了其他的属性，这是一次灾难。很容易就会忘记了修改这么多份副本中的其中一份。

其实style只是一系列属性的别名，如果我们这么定义我们的style会更容易：

```
<style name="MyButton">
    <item name="android:minWidth">@dimen/button_min_width</item>
    <item name="android:minHeight">@dimen/button_min_height</item>
</style>
```

现在你可以根据这个reference来修改一个属性值。去修改一个重复了无数次的layout的属性值无疑是荒谬的。你可以使用dimension来做到这一点，效果在style上完全一样。

我的意思并不是说你应该*总是*在style里使用resource references，只是说你应该在你需要成倍修改属性值的时候使用它。

这不是说有时候有时候你不需要通过资源限定符(resource qualifiers)重复一个style，你可以在最小限度内保持这一点。