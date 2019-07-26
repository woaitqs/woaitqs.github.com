---
title: '使用tools:namespace来方便预览'
layout: post
description: "使用tools:namespace来方便预览"
category: "android"
tags: [preview, tools, namespace]
---

在编写布局文件的时候，常常需要预览一下所写布局的样子。虽然 Android Studio 的 Preview 功能已经大大改进，但还是不能满足我们所有预览的需求，还有一些痛点亟待解决。

<!--more-->

1. 为预览时候编写的参数，无法自动在正式APK中自动失效。

2. RecyclerView 无法正确预览，也没有办法进行一些参数指定。

针对上诉痛点，其实官方早就给出了解决方案，提供了针对 Design-Time 相关的方案，只需要引入 tools 的命名空间即可。更多知识参考官方文档 -- [Tools Attributes Reference](https://developer.android.google.cn/studio/write/tool-attributes.html#toolssample_resources) 。

```xml
xmlns:tools="http://schemas.android.com/tools"
```

接下来看看，tools 是如何帮助我们解决预览时候的痛点的。

### 预览时候的 View 属性

在 XML 文件里，可以通过 `tools:{attribute}` 来设置 view 相关的属性，而这些属性将只会被`Android Studio layout editor`使用，实现了 Design-Time 与 Compile-Time 的分离，这也就意味着，通过这种方式制定的属性在打出来的包中是不生效的。

例如，一个显示姓名的 TextView，默认 text 属性为空，而我们可以通过设置 `tools:text=‘full-name’` ，这样在预览的时候就可以看到。又或者一个 VIP 标记View，默认不可见，但你又想知道其位置对不对，那就可以通过 `tools:visibility='visible'`, 在预览界面查看。

![tools_ns_example](http://o8p68x17d.bkt.clouddn.com/tools_ns_example.png)

我们常用的属性，在tools中都得到了支持，诸位可放心食用(好吃...)。

### RecyclerView 的特殊支持

在编写好一个 RecyclerView 的 XML 文件后，直接在预览查看时，会发现是 item0, item1, item2 的简单罗列，这显然不是我们想要的答案。tools 也提供了解决方案 -- `tools:listitem`。在下图可以看到，设置后的效果。

![tools_recycler_example](http://o8p68x17d.bkt.clouddn.com/tools_recycler_example.png)

除此之外，还有其他强大的功能哦。

1. itemCount 指定预览时，item 的数量。

2. listheader/listfooter 指定预览时，recycle 的头部/底部

### 预览自动生成的数据

有些时候，我们甚至不必去编辑 tools 中具体的值，Android Studio 提供了一些常见数据的 mock 值，通过 `"@tools:sample/*"` 进行指定。现在回过头看看上面的例子图片，有没有发现每个 item 的头像和名字都不一样了？

![tools_sample_data](http://o8p68x17d.bkt.clouddn.com/tools_sample_data.png)

在上图中，列举了一些官方提供的 sample data。

这还没完哦，Android Studio 甚至提供了自定义 SampleData 的入口，在项目目录中，右键 new 即可看到 `Sample Data Directory` 的选项。在进行自定义后，即可通过`文件名`进行调用了哦。由于界面介绍的知识已经大体上满足了我们 Preview 的需求了，这里就不进行展开，如果诸位有这方面需求的话，查看 [Tool Time](https://blog.stylingandroid.com/tool-time-part-2/) 这篇文章吧。

------------------------

### 文档信息
* 版权声明：自由转载-非商用-非衍生-保持署名（[创意共享3.0许可证](http://creativecommons.org/licenses/by-nc-nd/3.0/deed.zh)）
* 发表日期：2018年1月29日
* 社交媒体：[weibo.com/woaitqs](http://weibo.com/woaitqs)
* Feed订阅：[www.woaitqs.cc/feed.xml](http://www.woaitqs.cc/feed.xml)

------------------------