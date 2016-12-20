---
layout: post
title: "为什么是Gradle？ -- Gradle教程(一)"
description: "Gradle 系列教程"
category: "gradle"
keywords: "gradle, android, 教程"
tags: [android, gradle, build]
---
{% include JB/setup %}

开发 Android 的同时，也应该了解其构建构建工具。特别是在实际的生产开发项目中，对于需要分发的包有各式各样的需求，灵活应用构建工具将会极大地提升生产效率，另一方面，在日常开发中，对构建工具的了解，也能帮你省去不少额外工作。Gradle 他是你最亲密的战友之一，好好珍惜他。

<!--break-->

### Android 构建过程

Android 系统在构建的时候，需要将源码和资源文件打包进入APK中，然后进行签名，部署和分发。下图是对这个过程的简要说明，接下来具体说下执行的步骤。

![Android 打包过程](http://o8p68x17d.bkt.clouddn.com/build-process.png)

1. 编译器将源代码编译成 Dex 文件(Android 平台特定的格式)。
2. 打包器将上一步编译好的 Dex 文件和对应的资源文件打包到同一 APK 文件中。
3. 在这个 APK  部署到目标设备之前，必须要进行签名。如果是 Debug 版本，那么就会对应着有 Debug 版本的签名，通常情况下，这部分签名可以由 Android Studio 来提供。如果是 Release 版本的话，就需要提供对应的 Release 版本的 Keystore。
4. 在最后生成 APK 之前，Zipalign 会优化下 APK 的包结构，节省一点的空间。

### 为什么要用 Gradle

在前面的步骤里面可以看到，Android APK 打包的过程比较复杂，牵涉到的环节也非常地多。我们可以想象在打包过程中有哪些可以进行自定义的部分。例如多渠道、签名、打包类型等等太多了，当这些变量分子太多时，就需要一个足够强大的打包工具了。Gradle 就是其中的佼佼者。

Gradle 拥有如下的优点：

* 脚本语言，非常灵活，没有之一。
* 支持多 Project、多 Model 的配置，能够让层次更加鲜明。
* 非常强大的DSL (Domain Specific Language) ，领域相关语言，在 DSL 帮助下能帮我们省去很多额外工作。（例如 Android、Java 这些都是领域，DSL 在针对这些领域做的工作）。
* 采用了 Groovy 这个动态语言，相对 Ant、Maven 支持更多高阶属性。

当然 Android 采用 Gradle 的最主要原因是 Google 喜欢，233333.

![Gradle Logo](http://o8p68x17d.bkt.clouddn.com/gradle.jpg)

关于 Gradle 所采用的语言 Groovy，将在下一篇文章中做讲解，有兴趣的同学可以看看这篇文章。 [Learn Groovy in Y Minutes](https://learnxinyminutes.com/docs/groovy/)

可能有同学问为什么不直接讲 Gradle 的命令就好了？我的理由是，不会 Groovy，你无法深入了解 Gradle，对这个强大的构建工具将会始终流于表面。同样，Groovy 这种动态语言的编程范式，也会帮助大家开另一扇窗，看看外面更大更辽阔的世界。

------------------------

### 文档信息
* 版权声明：自由转载-非商用-非衍生-保持署名（[创意共享3.0许可证](http://creativecommons.org/licenses/by-nc-nd/3.0/deed.zh)）
* 发表日期：2016年12月20日
* 社交媒体：[weibo.com/woaitqs](http://weibo.com/woaitqs)
* Feed订阅：[www.woaitqs.cc/feed.xml](http://www.woaitqs.cc/feed.xml)

------------------------
