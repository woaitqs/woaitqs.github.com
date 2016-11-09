---
layout: post
title: "Android 启动卡顿原因查询"
description: "Android 启动卡顿原因查询"
keywords: "android, 卡顿"
category: "android"
tags: [卡顿, application]
---
{% include JB/setup %}

启动是一个什么样的过程？首先要对这个过程进行一个定义。考虑到进程是否存在，对启动时间有着明确的影响，将启动分为两种情况。

**冷启动** ：在进程不存在的情况下，从点击应用 Icon 到用户能看到界面所占用的时间。
**热启动** ：在进程依然存在的情况下，点击应用 Icon 到用户能看到对应的界面所用的时间。

<!--break-->

知道这两个启动时间怎么定义后，需要将这个进行实际的测量。通过这个测量时间的获得，就能定性地去衡量启动过程的性能问题。性能越好，所有时间就会更少，反之就更多。

### 宏观测量 — Adb 命令的使用

那么接下来看看，具体怎么将这个测量过程落地。

Android adb 提供了很简单的测量方式，预先通过在 FrameWork 层中的 ActivityManagerService 等等中进行埋点的方式，这样可以获得对应的时间点。

`adb shell am start -W [packageName]/[activityName]`

这样执行后，可以得到如下图所示的样子。

![am_start](http://o8p68x17d.bkt.clouddn.com/am_start.png)

这里面涉及到三个时间，ThisTime、TotalTime 和 WaitTime。WaitTime 是 startActivityAndWait 这个方法的调用耗时，ThisTime 是指调用过程中最后一个 Activity 启动时间到这个 Activity 的 startActivityAndWait 调用结束。TotalTime 是指调用过程中第一个 Activity 的启动时间到最后一个 Activity 的 startActivityAndWait 结束。如果过程中只有一个 Activity ，则 TotalTime 等于 ThisTime。

```java
final long startTime = SystemClock.uptimeMillis();
if (mWaitOption) {
    result = mAm.startActivityAndWait(null, null, intent, mimeType,
                null, null, 0, mStartFlags, profilerInfo, null, mUserId);
    res = result.result;
} else {
    res = mAm.startActivityAsUser(null, null, intent, mimeType,
            null, null, 0, mStartFlags, profilerInfo, null, mUserId);
}
final long endTime = SystemClock.uptimeMillis();
```

上面是 WaitTime 的计算方式，其他时间的计算方式也请参考源码。不过就实际的使用而言，这种方式得出的时间，不够准确，误差较大。

在 API 19 后，ActivityManagerService 也通过日志的方式告诉我们 Activity 启动花了多少时间。

> ActivityManager: Displayed com.android.myexample/.StartupTiming: +3s534ms

这个时间也可以作为高版本系统中的另一个宏观参考指标。

使用这种方式的话，不如在关键时间点记录相关的时间，这种方式会更加直观一点，粒度也会更加精细一些。

### 精确测量 — TraceView 的应用
大部分人都知道利用 TraceView 进行分析，通过 TraceView 提供的数据分析出对于哪些方法是造成耗时的缘故。这里还是要简单地介绍一下 TraceView，方便不了解的同学。

TraceView 位于 Android Devices Monitor 里面。可以在 Android Studio -》Tools -》Android 中找到 Devices Monitor。在 Monitor 中就可以打开对应的 Trace 文件，或者在这里进行 Trace 操作。下图展示的是一个 Trace 界面。

![TraceView](http://o8p68x17d.bkt.clouddn.com/traceview.png)

##### TraceView 关键指标的讲解
在上图中的下部分，是每个函数相关的调用信息。左边是函数的名称，右边是其调用的各种统计信息。
Incl Cpu Time: 这个方法调用所占据的时间。
Excl Cpu Time: 这个方法本身执行的时间，例如这个方法里面调用了 a, b 两个方法，Excl 就是排除 a，b 两者占用的时间之后的时间。
Incl Real Time 等等与前者类似，一个是 Real time，一个是 CPU 时间，两个区别不大。
Calls + Recur Calls / Total: 函数的调用次数(包含了递归调用) / 总次数。通常用于判断一些方法是否出现这个异常。
Cpu Time / Call： 每个函数的平均调用时间，这个对一些常用方法的判断尤为重要。例如一个 RecyclerView 经常调用 getview 方法，那么我对这个方法里面的优化是否生效呢？显然不能看单个 getview 的耗时，而是平均每个 getview 方法的耗时。

##### TraceView 的粗略使用指南
在 Android Devices Monitor 中左侧选中对应的进程名，然后点击如箭头所指的按钮，按钮中红点将会变成黑点，在界面上进行相应的操作，然后再次点击该按钮。等待片刻后，界面上就会出现相应的 TraceView 界面，根据上文所述的指标意义，找出问题的症结。

![Start Trace](http://o8p68x17d.bkt.clouddn.com/start_trace.png)

如果是针对启动问题的话，使用这种方式就不太合理了，还没等你点击按钮，就已经执行过了。给大家一种技巧，

```java
// 在 /sdcard 目录下记录 trace_file.trace 的文件, 开始 trace
Debug.startMethodTracing("trace_file");
// 结束 trace，并输出文件
Debug.stopMethodTracing();
```

在这种情况下，只要在启动的前后分别调用两个方法，就能在 sdcard 中找到对应的 trace 文件。后续再使用 adb pull 导出，并使用 Android Studio ，或者 Devices Monitor 打开文件，就能进行分析了。

### 官方的偷天换日

对于大型 APK 而言，优化并非那么简单，牵涉到很多模块，某些模块的初始化必须在此启动，在这种情况下，Google 官方教程推出了 [Launch-Time Performance | Android Developers](https://developer.android.com/topic/performance/launch-time.html) 这篇文章。这里简单地介绍下这种偷天换日的方案，让用户在体验上感觉应用快了那么一点点。

在启动 Activity 之间，系统会先生成一个空白的 Window，等到 Activity 的各类资源准备完毕后，将其放置到 Window 上去。偷天换日的方案就是在这个 window 上做手脚，先将 window 的背景替换成类似于主界面的背景，这样用户会以为此时界面已经在加载中了。等一段时间后，再将这个 Window 替换为实际的 Activity 界面。

首先指定一个背景图，就像下面的代码一样。

```xml
<layer-list xmlns:android="http://schemas.android.com/apk/res/android" android:opacity="opaque">
  <!-- The background color, preferably the same as your normal theme -->
  <item android:drawable="@android:color/white"/>
  <!-- Your product logo - 144dp color version of your app icon -->
  <item>
    <bitmap
      android:src="@drawable/product_logo_144dp"
      android:gravity="center"/>
  </item>
</layer-list>
```

再指定 Activity 的主题到这个 style 文件中。

```xml
<activity ...
  android:theme="@style/AppTheme.Launcher" />
```

最后，一定要记得在 Activity 的 onCreate 方法里进行 Style 替换。

```java
public class MyMainActivity extends AppCompatActivity {
  @Override
  protected void onCreate(Bundle savedInstanceState) {
     Make sure this is before calling super.onCreate
    setTheme(R.style.Theme_MyApp);
    super.onCreate(savedInstanceState);
     ...
  }
}
```

最好的方式还是优化代码结构，一劳永逸地解决问题，这样才能自豪地说，我就是快！

------------------------

### 文档信息
* 版权声明：自由转载-非商用-非衍生-保持署名（[创意共享3.0许可证](http://creativecommons.org/licenses/by-nc-nd/3.0/deed.zh)）
* 发表日期：2016年11月9日
* 社交媒体：[weibo.com/woaitqs](http://weibo.com/woaitqs)
* Feed订阅：[www.woaitqs.cc/feed.xml](http://www.woaitqs.cc/feed.xml)

------------------------
