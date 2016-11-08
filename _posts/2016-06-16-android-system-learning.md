---
layout: post
title: "Android 系统学习计划"
description: "本文是 Android 系统学习系列文章中的总纲，列举了打算解决的知识点，以及为什么要进行这个计划"
keywords: "android event 系统学习 handler binder 组件"
category: "android"
tags: [android, program, 系统学习]
---
{% include JB/setup %}

<div style="border:solid 1.5px #ccc;padding:20px 20px 10px 20px;margin-bottom: 20px;border-radius: 6px;">

          <span style="position: relative;padding: 5px 15px 5px 15px;background-color: white;bottom: 30px;font-size: 16px;font-weight: bolder;">摘要</span>

          <span style="position: relative;display: block;bottom: 15px;">
          本文是 Android 系统学习系列文章中的总纲，列举了打算解决的知识点，以及为什么要进行这个计划。对此系列感兴趣的同学，可以收藏这个链接 <a herf="http://www.woaitqs.cc/2016/06/16/android-system-learning.html">Android 系统学习</a>，也可以点击 <a href="http://www.woaitqs.cc/feed.xml">RSS订阅</a> 进行订阅。
          </span>
</div>

<!--break-->

------------------------------

### 一个想法

对历史有些了解的同学，应该都知道《海国图志》这本书，当时中国闭关锁国，社会、科技和经济发展严重滞后于其他国家，中国近代新思想的倡导者 [魏源](https://zh.wikipedia.org/zh-hk/%E9%AD%8F%E6%BA%90_(%E6%B8%85%E6%9C%9D)) 不满当时中国的现状，编撰了在当时对西方社会地理和历史最为详实的专著，也就是前文提及的[《海国图志》](https://book.douban.com/subject/1555922/) 。文中对 "蛮夷" 的理解有些偏袒，不过在当时之中国相当先进，这给愚昧的国人打开了一扇看外面世界的窗。这本书对我意义重大，以至于对于历史不好的我而言，忘却太多的历史珠玑，唯独这本书还记得，于我而言，就好比，一个科技工作者试图用自己的力量来改变国运，每每念及此，心里还是不免反复。

![意大利地图](https://ooo.0o0.ooo/2016/06/06/57555413eb4da.jpg)

![海国图志](https://ooo.0o0.ooo/2016/06/06/575549883edcf.jpg)

在经历一段时间的 Android 开发以后，虽然对各个方面都有涉及，还是没有形成很系统的知识，大抵都是散兵游勇，不成体系。在思考一番过后，决定建立一个对这些内容进行一次整理，以建立完整的知识结构，更便于自己对 Android 方面加深了解，进一步提升自己的内功。基于自己的认识，建立一个 Android 开发世界的 《海国图志》，也是对自己的提升。

本人能力有限，这些认识大多都是自己的一面之词，如果能侥幸帮到读者，就甚感欣慰。

----------------------

### 一个计划

在我所认知的 Android 开发中，可以大致分为如下几部分，分别如下：

- 系统服务，亦即系统提供的底层功能，主要涵盖 ActivityManagerServices 、WindowsServices 、SystemServer 等等。在进行应用开发时，免不了依赖于这些服务。如果把 Android 比喻成人，系统服务就是其灵魂，如果灵魂出现问题，那么肉身也难保。
- 应用组件，Android 是基于组件进行的开发系统，为我们所共知的是 Android 四大组件，Activity 、Services 、Broadcast 、ContentProvider。几乎所有 Android 程序都是由这 4 大组件通过 通信框架进行内部串联，并与系统服务进行通信，从而得到具有特定功能的应用。这部分就是人的肉体，颜面，决定了应用长什么样，有什么样的功能。
- 通信框架，这里主要是指进程间通信 和 线程间通信两大部分。进程间通信依附于 Binder Framework，而 Handler 则承担了一部分的线程间通信的工作。这两大模块，将整个系统串联起来，形成一个完整的整体，这就类似于人的骨架。
- 周边知识，例如打包、签名、插件化等等。这就是类似于人的装饰品，是给我们撑场面的。

因而这部分的海国图志，是关于上述四部分核心内容的归纳总结，拟定的目录如下：

#### 一、系统服务篇

1. <input type="checkbox" checked> [Android 如何启动？](http://www.woaitqs.cc/android/2016/06/15/how-android-launch-itself.html)
2. <input type="checkbox" checked> [Android 应用进程启动流程](http://www.woaitqs.cc/android/2016/06/21/activity-service.html)
3. <input type="checkbox" disabled=""> 什么是系统服务？
4. <input type="checkbox" disabled=""> ActivityManagerService
5. <input type="checkbox" disabled=""> SystemServer
6. <input type="checkbox" checked> [Android 应用安装过程源码解析](http://www.woaitqs.cc/android/2016/07/28/android-plugin-get-apk-info.html)
7. <input type="checkbox" disabled=""> WindowManagerService
8. <input type="checkbox" disabled=""> Zoyote 前世今生

#### 二、通信框架篇

1. <input type="checkbox" checked>  Binder 完全解析
    - <input type="checkbox" checked> [Binder 完全解析（一）概述](http://www.woaitqs.cc/android/2016/05/23/android-binder.html)
    - <input type="checkbox" checked>  [Binder 完全解析（二）设计详解](http://www.woaitqs.cc/android/2016/05/26/android-binder-token.html)
    - <input type="checkbox" checked>  [Binder 完全解析（三）AIDL实现原理分析](http://www.woaitqs.cc/android/2016/05/30/android-binder-proxy-and-token.html)
2. <input type="checkbox" checked>  Handler 通信框架
    - <input type="checkbox" checked>  [Handler消息机制源码解析](http://www.woaitqs.cc/android/2016/06/06/android-handler.html)

#### 三、应用组件篇

1. <input type="checkbox" disabled=""> Application 是什么？
2. <input type="checkbox" checked>  [Context 分析](http://www.woaitqs.cc/android/2016/09/07/android-context-implemention.html)
3. <input type="checkbox" disabled=""> Activity 组件分析
    - <input type="checkbox" checked>  [Activity生命周期是如何实现的](http://www.woaitqs.cc/android/2016/07/19/how-activity-lifecircle-work.html)
4. <input type="checkbox" checked> [Services 组件分析](http://www.woaitqs.cc/android/2016/09/20/android-service-usage.html)
5. <input type="checkbox" disabled=""> ContentProvider 组件分析
6. <input type="checkbox" disabled=""> Broadcast 组件分析

#### 四、珠玑拾遗

1. <input type="checkbox" disabled=""> Gradle 用法
2. <input type="checkbox" disabled=""> 混淆一二事
3. <input type="checkbox" checked>  Android View 全解析
    - <input type="checkbox" checked> [Android View 全解析(一) -- 窗口管理系统](http://www.woaitqs.cc/android/2016/10/10/android-view-theory-1.html)
    - <input type="checkbox" checked> [Android View 全解析(二) -- OnMeasure](http://www.woaitqs.cc/android/2016/10/18/android-view-theory-2.html)
    - <input type="checkbox" checked>  [Android View 全解析(三) -- onLayout](http://www.woaitqs.cc/android/2016/10/25/android-view-theory-3.html)
    - <input type="checkbox" checked>  [Android View 全解析(四) -- onDraw](http://www.woaitqs.cc/android/2016/10/28/android-view-theory-4.html)
    - <input type="checkbox" disabled=""> 自定义 View 动手实践
    - <input type="checkbox" disabled=""> 关于 View 事件分发

------------------------

### 一个愿望
这部分的工作势必会耗时较长的时间，期间也可能穿插其他内容，不过我会尽量保证进度，期待在完成时，能有一些不错的感悟和心得，最好是进入某种化境 ：）。

各位可以关注我的微博 [weibo.com/woaitqs](http://weibo.com/woaitqs)，以及个人网站 [woaitqs.cc](http://woaitqs.cc/)，有事可以邮件沟通 woaitqs[at]gmail.com。

感谢大家的支持！

------------------------

### 文档信息
* 版权声明：自由转载-非商用-非衍生-保持署名（[创意共享3.0许可证](http://creativecommons.org/licenses/by-nc-nd/3.0/deed.zh)）
* 发表日期： 2016年6月16日
* 社交媒体： [weibo](http://weibo.com/woaitqs)
* Feed订阅：[www.woaitqs.cc/feed.xml](http://www.woaitqs.cc/feed.xml)
