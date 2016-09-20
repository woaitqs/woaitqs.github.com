---
layout: post
title: "Android 事件处理机制分析"
description: "详细说明Android 事件处理机制分析"
keywords: "Touch, Android, Event, ViewGroup"
category: "android"
tags: [android, 事件处理, event]
---
{% include JB/setup %}


智能手机的操作都是通过各种事件来进行处理的，了解Android的事件处理机制，对日常的应用开发具有很多的好处。本篇文章将围绕事件处理机制进行展开，进行尽可能详实地分析，说明事件是如何在多个View层级之间进行传递的。

### 一、基础假定

由于篇幅的限定，我们只关注最基本的几种事件：Down、Move、Up和Cancel。一个Android定义的操作手势[Gestures](http://developer.android.com/training/gestures/index.html) 往往是由用户点击屏幕触发Down事件，用户移动手指会触发Move操作，操作结束的时候，会触发Up或者Cancel。可见对于这几个基本的事件的了解非常有助于我们学习Android的事件处理机制。同时为了叙述的方便，这里排除多点触摸的情况。

<!--break-->

### 二、处理事件概述

Android 中可见元素的基础单元就是View(ViewGroup)，事件处理也主要围绕着这两个类展开。用简明扼要的方法来说就是`从上往下传递，从下往上处理`，理解这一句话，就对Android的事件处理机制有了大体的认识。在ViewGroup中定义了几种基本方法，用来构建整个事件处理机制。

```java
// 事件派发
public boolean dispatchTouchEvent(MotionEvent ev)

// 事件拦截
public boolean onInterceptTouchEvent(MotionEvent ev)

// 事件消费
public boolean onTouchEvent(MotionEvent ev)
```

我们在说明下这几个函数的返回值代表的意义，这对于我们的理解非常重要。

> dispatchTouchEvent(MotionEvent ev)；

事件处理系统中的路由部分，`True`表示在当前View进行消费，并停止向下发放， `False`表明是指交由父亲View的`OnTouchEvent`来处理。

> onInterceptTouchEvent(MotionEvent ev);

事件处理系统的拦截器，`False`将事件放行到子View中，并交由子View的`onInterceptTouchEvent`来处理，`True`进行拦截并且交给当前View的`OnTouchEvent`来处理。

> onTouchEvent(MotionEvent ev);

事件处理系统的具体消费部分。`True`标明对此感兴趣，希望从此对其他事件感兴趣，并希望接受后面的事件；`False`则相反，标明对次不再感兴趣，不再接收后续的事件。即便对后续事件不感兴趣，在Action Down的时候，也应该Return True，以便能接受后续的事件。

### 三、事件分发流程

以下面的图为例，后续所说的内容将围绕这个图展开。
![View布局图](http://balpha.de/static/img/android-touch.png)

> ViewGroup事件分发流程

大多数的Android事件分发都是由`Activity`开始，而最后回到`Activity`结束的。当用户点击屏幕，硬件检测到Action Down事件，便开始由`Activity.dispatchTouchEvent()`方法进行下发，传递到DecorView上面去。

A.dispatchTouchEvent()的目的就是通过[hit testing algorithm](https://en.wikipedia.org/wiki/Hit-testing)算法找出那些包含点击区域的子View。再调用子View的dispatch方法之前，会先调用A.onInterceptTouchEvent()方法去看当前ViewGroup是否需要拦截事件然后自行处理(常见于Scroll事件)。onInterceptTouchEvent只可用在View Group中，默认实现为return false，即不进行拦截，调用子View的dispatch方法。如果return true，那么会发送ACTION_CANCEL到所有的子view去，所有后续的事件都会被A的onTouchListener或者onTouchEvent进行处理。同时A.onInterceptTouchEvent()不再会被调用。

这是ViewGroup进行拦截的例子
![ViewGroup进行拦截](http://codetheory.in/wp-content/uploads/2014/11/Android-Touch-System-Intercept-Example.png)

> View 事件处理

如果定义了View.onTouchListener，那么时间会先传递到这，如果未被消费（return true），那么View自身会通过View.onTouchEvent来进行处理。

如果View.onTouchListener返回ture，那么这将会变成所有后续事件的终点。如果return false，那么触摸事件将沿着View 层级从底往上传递到每一层。

换而言之，如果C.onTouchEvent返回false，那么这个事件处理流会交给B.onTouchEvent，如果B也返回false，那么会走到A.onTouchEvent。这里需要注意的是 Activity.onTouchEvent 是所有事件的终点，在这里return true和false已经没有意义。

### 四、处理事件相应流程图

![View布局图](http://2.bp.blogspot.com/-MoiWdrJYy4Y/UxrU8LWxrzI/AAAAAAAAAX4/Wf2Qwb0Xsc4/s1600/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7+2014-03-08+%E4%B8%8B%E5%8D%884.28.33.png)

### 五、参考资料
1. http://codetheory.in/understanding-android-input-touch-events/
2. http://balpha.de/2013/07/android-development-what-i-wish-i-had-known-earlier/
3. http://pierrchen.blogspot.hk/2014/03/pipeline-of-android-touch-event-handling.html
4. https://thenewcircle.com/s/post/1567/mastering_the_android_touch_system

-- EOF --
