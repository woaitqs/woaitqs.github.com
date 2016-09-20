---
layout: post
title: "自动检测性能问题 - BlockCanary 原理解析"
description: "Android BlockCanary 原理解析，自动检测泄露问题"
keywords: "android 插件 lifecycle 原理解析 binder activity"
category: "android"
tags: [android, program, performance]
---
{% include JB/setup %}

大家应该对 [LeakCanary](https://github.com/square/leakcanary) 比较熟悉，这个 Square 公司推出的自动检测内存泄露的工具，给开发人员带来了极大的便利。而国人也开发出一个用于自动检测性能卡顿的工具，[BlockCanary](https://github.com/markzhai/AndroidPerformanceMonitor), 今天将从原理上分析 BlockCanary 是如何实现的。

<!--break-->

### 怎么定义卡顿

介绍原理前，首先得知道什么是卡顿了？这就得引入视觉暂留这个概念。我们在看电影时，电影不是动态的，而是由一幕一幕播放完成的，但一般情况下，我们不会觉得卡顿，这就是视觉暂留的功效。

> 视觉暂留是光对视网膜所产生的视觉在光停止作用后，仍保留一段时间的现象，其具体应用是电影的拍摄和放映。原因是由视神经的反应速度造成的，其时值是二十四分之一秒。这也是是动画、电影等视觉媒体形成和传播的根据。  

简单地说，如果相隔的两帧间隔不超过二十四分之一秒, 人是无法感知的。对于 Android 手机而言，如果屏幕之间的变化时间足够短，那我们就不会觉得卡顿。

Android 系统在 Kitkat 之后，也引入了黄油计划，这个计划目的在于优化 Android 的各种性能体验。在这其中，最重要的功能是引入了 VSYNC。VSYNC(垂直同步信号) 会按照 60 FPS/second 的频率，也就是差不多 16.67ms 的间隔，进行一次信号发送，而 Android 的绘制机制会根据这个信号来进行相应的处理。如果一次页面渲染未能在 16.67ms 内完成，那么界面就不会发生改变，只能等待下一次 VSYNC 信号，然而这样就会照成卡顿。

所以，结合上面粗略的知识介绍，卡顿可以在一定程度上，通过统计两帧之间间隔大于 16.67ms 的次数，来进行定量的衡量。

关于 Vsync 的更多知识，以及黄油计划的信息，可以参考这个链接 [http://www.androidpolice.com/2012/07/12/getting-to-know-android-4-1-part-3-project-butter-how-it-works-and-what-it-added/](http://www.androidpolice.com/2012/07/12/getting-to-know-android-4-1-part-3-project-butter-how-it-works-and-what-it-added/)。

### Looper 提供的机制

先看看我们熟悉的 Looper 的源码，里面实现的功能就是不断地从 MessageQueue 里面取出 Message 对象，并加以执行。

```java
for (;;) {
    Message msg = queue.next(); // might block
    if (msg == null) {
        // No message indicates that the message queue is quitting.
        return;
    }

    // This must be in a local variable, in case a UI event sets the logger
    Printer logging = me.mLogging;
    if (logging != null) {
        logging.println(">>>>> Dispatching to " + msg.target + " " +
                msg.callback + ": " + msg.what);
    }

    msg.target.dispatchMessage(msg);

    if (logging != null) {
        logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
    }

    // ignore some code...

    msg.recycleUnchecked();
}
```

注意到，在 `dispatchMessage` 的前后，分别有两个 log 的输出事件，而 `dispatchMessage` 就是线程上的一次消息处理。如果两次消息处理事件，都超过了 16.67ms, 那就一定发生了卡顿，这也是 BlockCanary 的基础原理。

BlockCanary 实现了 Printer，我们看看具体的实现。

```java
class LooperMonitor implements Printer {
  @Override
  public void println(String x) {
      if (!mPrintingStarted) {
          mStartTimestamp = System.currentTimeMillis();
          mStartThreadTimestamp = SystemClock.currentThreadTimeMillis();
          mPrintingStarted = true;
          startDump();
      } else {
          final long endTime = System.currentTimeMillis();
          mPrintingStarted = false;
          if (isBlock(endTime)) {
              notifyBlockEvent(endTime);
          }
          stopDump();
      }
  }

  private boolean isBlock(long endTime) {
      return endTime - mStartTimestamp > mBlockThresholdMillis;
  }

  // ignore other codes...
}
```

这里实现了 println 方法，而 Looper 中也调用了 println 方法，而且在除非 dump 日志的情况下，也只有在事件消息前后进行 println 操作。换而言之，我们可以初步认为两个 println 调用之间的时间超过 16.67ms 就证明了卡顿。上面的代码也非常地清晰明了说明了这点。

### 输出堆栈

在发现卡顿时，还需要提供当前线程的堆栈，这样才能方便开发人员知晓在哪里发生了卡顿，而 Java 刚好也提供了类似的机制，代码也非常的简单。

```java
@Override
protected void doSample() {
    StringBuilder stringBuilder = new StringBuilder();

    for (StackTraceElement stackTraceElement : mCurrentThread.getStackTrace()) {
        stringBuilder
                .append(stackTraceElement.toString())
                .append(BlockInfo.SEPARATOR);
    }

    synchronized (sStackMap) {
        if (sStackMap.size() == mMaxEntryCount && mMaxEntryCount > 0) {
            sStackMap.remove(sStackMap.keySet().iterator().next());
        }
        sStackMap.put(System.currentTimeMillis(), stringBuilder.toString());
    }
}
```

BlockCanary 的原理简介就到这里，还是比较轻松和简单的。

### Choreographer

最近在腾讯的博文中，也看到了另一种更为精确的方案，这里做简单介绍。方案上采用了同样来自黄油计划的 Choreographer。

在其源码中，有这样的一个接口 FrameCallback.

```java
public interface FrameCallback {

    // 每帧绘制的时候的回调
    public void doFrame(long frameTimeNanos);
}
```

```java
public void postFrameCallback(FrameCallback callback) {
    postFrameCallbackDelayed(callback, 0);
}
```

如上，Choreographer 提供了注入 FrameCallback 的入口。我们只需要对两次 `doFrame` 的时间间隔进行计算，超过 16.67ms 就算卡顿。

但根据作者的统计，这种方法比较难拿到堆栈消息，只适合在线上进行量化的统计，不方便进行卡顿原因的分析。

------------------------

### 文档信息
* 版权声明：自由转载-非商用-非衍生-保持署名（[创意共享3.0许可证](http://creativecommons.org/licenses/by-nc-nd/3.0/deed.zh)）
* 发表日期：2016年9月1日
* 社交媒体：[weibo.com/woaitqs](http://weibo.com/woaitqs)
* Feed订阅：[www.woaitqs.cc/feed.xml](http://www.woaitqs.cc/feed.xml)

------------------------
