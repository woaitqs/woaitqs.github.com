---
layout: post
title: "常见滑动冲突解决方案"
description: "经常出现滑动、触碰冲突的情况，应该如何解决了？"
category: "android"
keywords: "RecyclerView, ViewPager, Touch"
tags: [android,RecyclerView,ViewPager]
---
{% include JB/setup %}

在看这篇文章之前，需要对 Android 的事件分发机制有足够详细的了解，链接中的文章 ![Android 事件处理机制分析](http://www.woaitqs.cc/android/2016/03/05/android-touch-system) 会对大家理解事件分发机制有所帮助。

废话不多说，进入正题。我们在日常开发中，不是会需要自己进行事件处理。如果产品有一些反人类的设计，你还没法说服它的时候，
