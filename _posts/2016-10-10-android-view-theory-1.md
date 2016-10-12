---
layout: post
title: "Android View 全解析(一) -- 窗口管理系统"
description: "android view 的窗口管理系统"
keywords: "窗口管理, view"
category: "android"
tags: [view, window, 窗口管理]
---
{% include JB/setup %}

一直在写 Android Framework 层的东西，虽然很重要，但一直写这方面的内容，还是有些不够接地气。思前想后一番后，打算写写关于 View 这个在 Android 中平常得不能更平常的东西。由浅入深，仔细梳理，帮助我，也希望能帮助你，更好地再次认识 View。

第一篇文章主要讲 Android 的窗口管理系统，依托于这套系统，我们才能将 View 显示到屏幕上。了解这套系统，有助于更好地理解 Android View 的来龙去脉。但这个系统复杂，涉及到的方面众多，在这里也只是简述，还望见谅。

<!--break-->

### 窗口管理系统大家族

这个小节，主要说一下家族里面的成员，分别是 `view`,`window`,`windowmanager`,`viewRoot`。小节里面主要介绍它们是什么，以及互相之间的关系是什么。

在谈及 View 之前，必须得说说 Window 这个东西，通常我们认为显示在界面上的是 View，这么说本身没有什么问题，但更准确的说法是 WindowManager 通过 ViewRoot 将 View 和 Window 协同整合在一起，最终将 View 展示在 Window 上面。正如 Window 这个名字，就是窗口的意思，我们所见的所有东西都要展示在 Window 上，熟知的 Dialog、Activity 以及 Toast 都是展示在 Window 上面的。

Window 本身是一个抽象类，提供了对标准 UI 行为的一些支持，例如背景、标题栏和按键等等。我们使用的是它的子类，也是唯一的实现 PhoneWindow。

Window的显示有多种类型，具体可以在这里看到 [https://developer.android.com/reference/android/view/WindowManager.LayoutParams.html](https://developer.android.com/reference/android/view/WindowManager.LayoutParams.html)。普通开发的应用，就是类型为 TYPE_APPLICATION 的 Window。最近一些工具应用开始使用悬浮窗，特别是一些手机清理软件，悬浮窗通常采用的是 TYPE_SYSTEM_ALERT、TYPE_PHONE, 对于 API 在 19 以上的系统，使用的是 TYPE_TOAST。

成功的男人背后都有一个伟大的女人，一个成功的 Window 背后就有一个伟大的 WindowMananger。WindowManager 本身是一个接口，提供了与 Window 交互的基础功能，分别是添加、更新和删除 View 的接口。

```java
public interface ViewManager {
    public void addView(View view, ViewGroup.LayoutParams params);
    public void updateViewLayout(View view, ViewGroup.LayoutParams params);
    public void removeView(View view);
}
```

WindowManager 的实现沿用了 C/S 结构，WindowManager 只作为一个代理，实际工作的是 WindowManagerService。WindowManangerService 以 Session 的形式来管理各个 Application 的窗口，系统启动了多少个含有 View 的应用，就有多少个对应的 Session。下面的图，说明了这一点。

![图片来自 www.programgo.com](http://o8p68x17d.bkt.clouddn.com/window_session.jpg)

一个好汉三个帮，View 系统也不例外，WindowManager 与 View 之间不直接进行交互，而是依托于一个中间商，叫做 ViewRoot。ViewRoot 本身是一个 Handler，通过 ViewRoot 实现了两者间的消息传递。ViewRoot 将通知 View 进行相应的界面绘制，然后调用 WindowManager 提供的接口，将 View 添加或更新到 Window 上面。

### 添加 DecorView 到 PhoneWindow

我们以 Activity 的 setContentView 入手，看看窗口管理系统是如何介入这个过程的。

```java
public void setContentView(@LayoutRes int layoutResID) {
    getWindow().setContentView(layoutResID);
    initWindowDecorActionBar();
}
```

这里调用了 getWindow 方法，需要解释的是对于每个 Activity 而言都有一个对应的窗口，就是前文提及的 PhoneWindow，这个 PhoneWindow 有一个 View ，叫做 DecorView，这也是界面布局的根节点。随便找了一个 App 的界面，大家可以注意箭头标示的地方，那里就是 DecorView。

![decor view](http://o8p68x17d.bkt.clouddn.com/decor_view.png)

setContentView 的实现在其子类 PhoneWindow 中，上面的图也可以看到。

```java
@Override
public void setContentView(int layoutResID) {
    // Note: FEATURE_CONTENT_TRANSITIONS may be set in the process of installing the window
    // decor, when theme attributes and the like are crystalized. Do not check the feature
    // before this happens.
    if (mContentParent == null) {
        installDecor();
    } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        mContentParent.removeAllViews();
    }

    if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                getContext());
        transitionTo(newScene);
    } else {
        mLayoutInflater.inflate(layoutResID, mContentParent);
    }
    mContentParent.requestApplyInsets();
    final Callback cb = getCallback();
    if (cb != null && !isDestroyed()) {
        cb.onContentChanged();
    }
}
```

重点在于 installDecor 方法中，这里是给 Window 添加根 View，也就是 DecorView。

```java
private void installDecor() {
    if (mDecor == null) {
        mDecor = generateDecor();
        mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
        mDecor.setIsRootNamespace(true);
        if (!mInvalidatePanelMenuPosted && mInvalidatePanelMenuFeatures != 0) {
            mDecor.postOnAnimation(mInvalidatePanelMenuRunnable);
        }
    }
    if (mContentParent == null) {
        mContentParent = generateLayout(mDecor);
        // ...
    }

    // ...

}
```

在这个方法中完成了 Transaction 动画的设置，标题栏的配置，decor 背景的设置等等初始化工作。代码相对比较繁琐，有兴趣的同学，可以看看源码中 PhoneWindow 的实现。

在这之后，View 就可以显示了吗？大家可能也不会觉得是这么简单的，毕竟我们熟知的 onMeasure、onLayout 和 onDraw 都还没有调用呢？看起来，我们需要一个时机来触发 View 的操作。在下一小节，就来说一下什么时候触发这一过程。

### View 什么时候开始绘制

Android 系统在 4.0 后为更好的用户体验，执行了一个「黄油计划」，而其中最重要的部分就是垂直同步机制(VSYNC)。让我们首先在大体上理解一下 VSYNC。

> VSync stands for Vertical Synchronization. The basic idea is that synchronizes your FPS with your monitor's refresh rate. The purpose is to eliminate something called "tearing". I will describe all these things here. Every CRT monitor has a refresh rate.

这是从一篇论文里面引用过来的，VSYNC 就是一种同步机制，以某种固定的频率进行同步，当其他组件收到这个同步信号时，就执行相应的操作。设想一下，如果没有这个同步机制，各个模块又怎能知道在哪个时候去执行自己的工作了？ 这里可以初步地将 VSYNC 当做闹钟，每间隔固定时间，就响一次，其他组件听到闹铃后，就开始干活了。这个间隔的时间，与屏幕刷新频率有关，例如大多数 Android 设备的刷新频率是 60 FPS(Frame per second)，一秒钟刷新60次，因而间隔时间就是 1000 / 60 = 16.667 ms。这个时间，大家是不是很熟悉了？看过太多性能优化的文章，都说每一帧的绘制时间不要超过 16 ms，其背后的原因就是这个。绘制每一帧对应的 View，这个步骤发生在 UI 线程上，所以也不要在 UI 线程上进行耗时的操作，否则就可能在 16 ms内，无法完成界面更新操作了。

![VSYNC](http://o8p68x17d.bkt.clouddn.com/vsync.jpg)

图中中间部分，显示了试用 VSYNC 技术后的显示效果，可以看到整体上，效果要优于其他方案。

为了将 VSYNC 机制在 Framework 落地下来，在 View 层引入了 [Choreographer](https://developer.android.com/reference/android/view/Choreographer.html)。这个类的主要责任就是协调动画、输入和绘制，使得用户体验上达到一致的效果。通常情况下，我们不用去理会这个 Choreographer，Android Framework 通过更高层级的抽象，帮我们完成了这一步骤，他们分别实现在 Animation，onDraw 等代码中。

下面大体上，看看 Choreographer 的实现。

```java
private static final ThreadLocal<Choreographer> sThreadInstance =
        new ThreadLocal<Choreographer>() {
    @Override
    protected Choreographer initialValue() {
        Looper looper = Looper.myLooper();
        if (looper == null) {
            throw new IllegalStateException("The current thread must have a looper!");
        }
        return new Choreographer(looper);
    }
};
```

这里采用了 ThreadLocal 这个关键字，使用这个关键字后，能够保证对于单个线程，只能获取到同一实例。而 Choreographer 在构建的时候，还要求这个线程必须有 Looper，Choreographer 需要利用 Looper 来进行相应的消息通知。再看看构造函数的实现。

```java
private Choreographer(Looper looper) {
    mLooper = looper;
    mHandler = new FrameHandler(looper);
    mDisplayEventReceiver = USE_VSYNC ? new FrameDisplayEventReceiver(looper) : null;
    mLastFrameTimeNanos = Long.MIN_VALUE;

    mFrameIntervalNanos = (long)(1000000000 / getRefreshRate());

    mCallbackQueues = new CallbackQueue[CALLBACK_LAST + 1];
    for (int i = 0; i <= CALLBACK_LAST; i++) {
        mCallbackQueues[i] = new CallbackQueue();
    }
}
```

新建了一个 Handler 用来做消息通信，其后建立了一个 CallbackQueue，这里的 CallbackQueue 主要有四种形式，当收到 VSYNC 信号后，依次执行其中的 Callback。

```java
try {
    Trace.traceBegin(Trace.TRACE_TAG_VIEW, "Choreographer#doFrame");

    mFrameInfo.markInputHandlingStart();
    doCallbacks(Choreographer.CALLBACK_INPUT, frameTimeNanos);

    mFrameInfo.markAnimationsStart();
    doCallbacks(Choreographer.CALLBACK_ANIMATION, frameTimeNanos);

    mFrameInfo.markPerformTraversalsStart();
    doCallbacks(Choreographer.CALLBACK_TRAVERSAL, frameTimeNanos);

    doCallbacks(Choreographer.CALLBACK_COMMIT, frameTimeNanos);
} finally {
    Trace.traceEnd(Trace.TRACE_TAG_VIEW);
}
```

这里的 CALLBACK_TRAVERSAL 就是与 View 绘制相关的部分。在 ViewRootImpl 代码中，也可以证实到这一点。

```java
void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        if (!mUnbufferedInputDispatch) {
            scheduleConsumeBatchedInput();
        }
        notifyRendererOfFramePending();
        pokeDrawLockIfNeeded();
    }
}
```

这里的 mTraversalRunnable 实现非常简单。

```java
final class TraversalRunnable implements Runnable {
    @Override
    public void run() {
        doTraversal();
    }
}
```

而从 doTraversal 开始，View 就开始真正进行绘制了。在 Choreographer 的英明领导下，View 开始准备开工干活了。

这里和前面的知识进行下串联，再进行下总结。

> 当 Choreographer 接收到 VSYNC 信号后，ViewRootImpl 调用 scheduleTraversals 方法，通知 View 进行相应的渲染，其后 ViewRootImpl 将 View 添加或更新到 Window 上去。

在 doTraversal 方法中，会调用到 performTraversals 方法，这是一个巨长的方法，在 API-23 版本中，代码行数达到了800行。我们绝大多数的类，都没有达到这个数量级 ：）。在这个方法里，就会执行到我们熟悉的 View 三部曲，Measure、Layout 和 Draw。

在后续的文章中，再来详细说明 View 中的 Measure、Layout 和 Draw 是如何实现的。再会 ：）

------------------------

### 文档信息
* 版权声明：自由转载-非商用-非衍生-保持署名（[创意共享3.0许可证](http://creativecommons.org/licenses/by-nc-nd/3.0/deed.zh)）
* 发表日期：2016年10月10日
* 社交媒体：[weibo.com/woaitqs](http://weibo.com/woaitqs)
* Feed订阅：[www.woaitqs.cc/feed.xml](http://www.woaitqs.cc/feed.xml)

------------------------
