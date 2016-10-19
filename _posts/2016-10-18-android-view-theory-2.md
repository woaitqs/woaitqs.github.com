---
layout: post
title: "Android View 全解析(二) -- OnMeasure"
description: "android view OnMeasure 的执行过程"
keywords: "OnMeasure, view"
category: "android"
tags: [view, window, OnMeasure]
---
{% include JB/setup %}

在[上篇文章](http://www.woaitqs.cc/android/2016/10/10/android-view-theory-1.html)中，介绍了 view 与窗口系统的关系，以及在这个系统中是怎么触发 View 的三类重要事件的。接下来说说，三类事件中 onMeasure 事件，并以 FrameLayout 的 onMeasure 为例详细说明 measure 过程是如何进行的。

<!--break-->

------------------------

### MeasureSpec 的定义与实现

首先先看看 performTraversals 里是如何进行 Measure 过程的。由于这个方法代码繁多，这里只展示与 Measure 过程相关的代码。

```java
int childWidthMeasureSpec = getRootMeasureSpec(mWidth, lp.width);
int childHeightMeasureSpec = getRootMeasureSpec(mHeight, lp.height);

performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
```

首先说明一下这里的 MeasureSpec 是什么，这里是利用了位运算，将一个 int 类型包含了两种信息，分别是 size 和 mode。java 的 int 类型，可以表示 32 位数字，最高的两位数字用来表示 mode，其余的部分用来表示 size。对于手机屏幕而言，size 一般都优先，不用担心需要 31 位数字来表达 size 的情况。

MeasureSpec 分别有三种 mode，分别是 UNSPECIFIED, EXACTLY 和 AT_MOST.

```java
/**
 * Measure specification mode: The parent has not imposed any constraint
 * on the child. It can be whatever size it wants.
 */
public static final int UNSPECIFIED = 0 << MODE_SHIFT;

/**
 * Measure specification mode: The parent has determined an exact size
 * for the child. The child is going to be given those bounds regardless
 * of how big it wants to be.
 */
public static final int EXACTLY     = 1 << MODE_SHIFT;

/**
 * Measure specification mode: The child can be as large as it wants up
 * to the specified size.
 */
public static final int AT_MOST     = 2 << MODE_SHIFT;
```

UNSPECIFIED: 标明自身对 View 大小没有任何限制，需要子 View 的信息来帮助协定

EXACTLY: View 已经确定自身的大小

AT_MOST: 父 View 已经限定了最大大小，具体 View 能不能超过这个限制，得看不同 View 的实现情况。

而对于 Size 而言，就是具体的数值大小了。Android 提供了生成 MeasureSpec 的静态方法。

```java
public static int makeMeasureSpec(int size, int mode) {
    if (sUseBrokenMakeMeasureSpec) {
        return size + mode;
    } else {
        return (size & ~MODE_MASK) | (mode & MODE_MASK);
    }
}
```

至此，我们知道了 MeasureSpec 可以用来表征一个 View 的大小信息，接下来的章节看看，这些信息是如何应用的。

------------------------

### DecorView 的 MeasureSpec

回到最开始的代码片段，我们看看 getRootMeasureSpec 是如何实现的，通过这个方法我们就可以知道 DecorView 是如何定义自己的大小的。

```java
private static int getRootMeasureSpec(int windowSize, int rootDimension) {
    int measureSpec;
    switch (rootDimension) {

    case ViewGroup.LayoutParams.MATCH_PARENT:
        // Window can't resize. Force root view to be windowSize.
        measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.EXACTLY);
        break;
    case ViewGroup.LayoutParams.WRAP_CONTENT:
        // Window can resize. Set max size for root view.
        measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.AT_MOST);
        break;
    default:
        // Window wants to be an exact size. Force root view to be that size.
        measureSpec = MeasureSpec.makeMeasureSpec(rootDimension, MeasureSpec.EXACTLY);
        break;
    }
    return measureSpec;
}
```

从上面的代码中，我们可以看到 DecorView 的大小是与 Window 的大小和 layoutParams 相关的，如果是 MATCH_PARENT, 那么 DecorView 的 MeasureSpec 就是精确的 Window 大小。接下来看看 windowSize， rootDimension 是怎么赋值的？我这里直接贴代码。

```java
final WindowManager.LayoutParams mWindowAttributes = new WindowManager.LayoutParams();

final WindowManager.LayoutParams lp = mWindowAttributes;

public LayoutParams() {
    super(LayoutParams.MATCH_PARENT, LayoutParams.MATCH_PARENT);
    type = TYPE_APPLICATION;
    format = PixelFormat.OPAQUE;
}
```

从这里可以看到默认情况下，window 的 LayoutParams 就是 MATCH_PARENT, 这样 DecorView 的大小就是 Window 的大小。

------------------------

### View 的 Measure 实现

接着看最开始代码中 performMeasure 的实现。

```java
private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
    Trace.traceBegin(Trace.TRACE_TAG_VIEW, "measure");
    try {
        mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
}
```

代码中执行到了 View 的 Measure 方法里面。注意到下面的代码签名，使用了 final 关键字，这样任何继承自 View 的类都不能复写这个方法，因为在这个类中涉及到了 Measure 的重要流程，如果这个过程被打断，那么会导致整个流程失败，因而这里添加 final 关键字是很有必要的。代码中还省略了部分与缓存相关的代码，代码中的重点在于 `onMeasure` 这个方法，一定要在这个方法中，指定 MeasureSpec，否则就不能知道 view 的大小了，后续的步骤也不能继续。

```java
public final void measure(int widthMeasureSpec, int heightMeasureSpec) {

        // Suppress sign extension for the low bytes
        long key = (long) widthMeasureSpec << 32 | (long) heightMeasureSpec & 0xffffffffL;

        if ((mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT ||
                widthMeasureSpec != mOldWidthMeasureSpec ||
                heightMeasureSpec != mOldHeightMeasureSpec) {

            int cacheIndex = (mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT ? -1 :
                    mMeasureCache.indexOfKey(key);
            if (cacheIndex < 0 || sIgnoreMeasureCache) {
                // measure ourselves, this should set the measured dimension flag back
                onMeasure(widthMeasureSpec, heightMeasureSpec);
                mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
            }

            // flag not set, setMeasuredDimension() was not invoked, we raise
            // an exception to warn the developer
            if ((mPrivateFlags & PFLAG_MEASURED_DIMENSION_SET) != PFLAG_MEASURED_DIMENSION_SET) {
                throw new IllegalStateException("View with id " + getId() + ": "
                        + getClass().getName() + "#onMeasure() did not set the"
                        + " measured dimension by calling"
                        + " setMeasuredDimension()");
            }

            mPrivateFlags |= PFLAG_LAYOUT_REQUIRED;
        }

    }
```

我们先看看 onMeasure 的默认实现。

```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
            getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
}

public static int getDefaultSize(int size, int measureSpec) {
    int result = size;
    int specMode = MeasureSpec.getMode(measureSpec);
    int specSize = MeasureSpec.getSize(measureSpec);

    switch (specMode) {
    case MeasureSpec.UNSPECIFIED:
        result = size;
        break;
    case MeasureSpec.AT_MOST:
    case MeasureSpec.EXACTLY:
        result = specSize;
        break;
    }
    return result;
}
```

默认实现而言，如果 View 的 mode 仍旧为 UNSPECIFIED，将其指定为最小size(可以人为指定，否则为0，或者背景的最小值)，否则为 measureSpec 中指定的大小。

------------------------

### ViewGroup 的 Measure 实现

相对复杂的是 ViewGroup，ViewGroup 继承自 View，但 View 的 measure 方法是 final 的，ViewGroup 需要自定义 measure 过程时就得复写 onMeasure 方法。每个 ViewGroup 的需求不同，很难统一起来讲，这里以 FrameLayout 为例，讲述它的 Measure 过程，毕竟这是几个官方 ViewGroup 中最简单的一个。

```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    int count = getChildCount();

    // 是否高或者宽位置
    final boolean measureMatchParentChildren =
            MeasureSpec.getMode(widthMeasureSpec) != MeasureSpec.EXACTLY ||
            MeasureSpec.getMode(heightMeasureSpec) != MeasureSpec.EXACTLY;
    // 清除原有保留的对象
    mMatchParentChildren.clear();

    int maxHeight = 0;
    int maxWidth = 0;
    int childState = 0;

    // 遍历每一个 view
    for (int i = 0; i < count; i++) {
        final View child = getChildAt(i);
        // 如果高宽位置，或者 child 是可见的，那么就需要对 child 进行 measure
        if (mMeasureAllChildren || child.getVisibility() != GONE) {
            // 调用这个方法时，会让 child 自身进行测量
            // 参数中的两个 0，表示父 Parent 已经占用的空间，这里为 0
            // widthMeasureSpec 和 heightMeasureSpec 分别是父 Parent 的 高宽 MeasureSpec
            measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, 0);
            // 经过 Measure 过程后，就能知道自己的 LayoutParams
            // LayoutParams 含有了自己的宽高信息
            final LayoutParams lp = (LayoutParams) child.getLayoutParams();
            // 更新最大宽度
            maxWidth = Math.max(maxWidth,
                    child.getMeasuredWidth() + lp.leftMargin + lp.rightMargin);
            // 更新最大高度
            maxHeight = Math.max(maxHeight,
                    child.getMeasuredHeight() + lp.topMargin + lp.bottomMargin);
            // 不是很懂这句话... 逃 ：）
            childState = combineMeasuredStates(childState, child.getMeasuredState());
            if (measureMatchParentChildren) {
                if (lp.width == LayoutParams.MATCH_PARENT ||
                        lp.height == LayoutParams.MATCH_PARENT) {
                    // 要对 MatchParent 的做特殊处理，下面的代码可以看到
                    mMatchParentChildren.add(child);
                }
            }
        }
    }

    // Account for padding too
    // 加上前景的上下左右 padding
    maxWidth += getPaddingLeftWithForeground() + getPaddingRightWithForeground();
    maxHeight += getPaddingTopWithForeground() + getPaddingBottomWithForeground();

    // Check against our minimum height and width
    // 与设定的最大和最小做校验
    maxHeight = Math.max(maxHeight, getSuggestedMinimumHeight());
    maxWidth = Math.max(maxWidth, getSuggestedMinimumWidth());

    // Check against our foreground's minimum height and width
    // 前景的最小高宽
    final Drawable drawable = getForeground();
    if (drawable != null) {
        maxHeight = Math.max(maxHeight, drawable.getMinimumHeight());
        maxWidth = Math.max(maxWidth, drawable.getMinimumWidth());
    }

    // 设置自身的 measureSpec
    // resolveSizeAndState 是 view 提供的静态方法，返回对应的 measureSpec
    setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState),
            resolveSizeAndState(maxHeight, heightMeasureSpec,
                    childState << MEASURED_HEIGHT_STATE_SHIFT));

    // 处理每个 LayoutParams 中含有 MATCH_PARENT 的情况。
    count = mMatchParentChildren.size();
    if (count > 1) {
        for (int i = 0; i < count; i++) {
            final View child = mMatchParentChildren.get(i);
            final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

            final int childWidthMeasureSpec;
            if (lp.width == LayoutParams.MATCH_PARENT) {
                // // MATCH_PARENT 的宽度设置为最大宽度
                final int width = Math.max(0, getMeasuredWidth()
                        - getPaddingLeftWithForeground() - getPaddingRightWithForeground()
                        - lp.leftMargin - lp.rightMargin);
                childWidthMeasureSpec = MeasureSpec.makeMeasureSpec(
                        width, MeasureSpec.EXACTLY);
            } else {
                // 返回其默认的情况
                childWidthMeasureSpec = getChildMeasureSpec(widthMeasureSpec,
                        getPaddingLeftWithForeground() + getPaddingRightWithForeground() +
                        lp.leftMargin + lp.rightMargin,
                        lp.width);
            }

            final int childHeightMeasureSpec;
            if (lp.height == LayoutParams.MATCH_PARENT) {
                final int height = Math.max(0, getMeasuredHeight()
                        - getPaddingTopWithForeground() - getPaddingBottomWithForeground()
                        - lp.topMargin - lp.bottomMargin);
                childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(
                        height, MeasureSpec.EXACTLY);
            } else {
                childHeightMeasureSpec = getChildMeasureSpec(heightMeasureSpec,
                        getPaddingTopWithForeground() + getPaddingBottomWithForeground() +
                        lp.topMargin + lp.bottomMargin,
                        lp.height);
            }

            child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
        }
    }
}
```

大部分代码都通过注释精选了说明，这里就自定义 View 中常用的几个 static 方法进行简单的说明，方便读者有一个大题的认识。

> `measureChildWithMargins`
要求 child 进行 Measure，在绘制的时候将自己的 padding 和 margins 考虑进去。经过 Measure 过后，可以得到对应的 LayoutParams
这个方法要求 child 必须得是 MarginLayoutParams，大部分的容器 View (LinearLayout、FrameLayout等等)都是这个 MarginLayoutParams.

> `measureChild`
这个方法与前面这个是相对的，不同之处在于只考虑自身的 padding，不考虑 margin。

> `measureChildren`
对所有可见 view 调用 measureChild 方法。

> `getChildMeasureSpec`
这个是上述方法都会调用的方法，分别有三个参数，当前view的measureSpec, 当前view 的 padding(或者加上margin), 当前 view 期望的大小(不一定会实现)。
方法根据特定的规则来返回对应的 MeasureSpec，这是蔚为重要的一个方法。对应的规则简单如下：

![图片来自 http://blog.qiji.tech/archives/7740](http://o8p68x17d.bkt.clouddn.com/measure.jpg)

------------------------

### 参考文献：
1. [http://ezio.farbox.com/post/android-view-measure-1](http://ezio.farbox.com/post/android-view-measure-1)
2. [http://www.liaohuqiu.net/posts/how-does-android-caculate-the-size-of-child-view/](http://www.liaohuqiu.net/posts/how-does-android-caculate-the-size-of-child-view/)
3. [http://blog.qiji.tech/archives/7740](http://blog.qiji.tech/archives/7740)

------------------------

### 文档信息
* 版权声明：自由转载-非商用-非衍生-保持署名（[创意共享3.0许可证](http://creativecommons.org/licenses/by-nc-nd/3.0/deed.zh)）
* 发表日期：2016年10月10日
* 社交媒体：[weibo.com/woaitqs](http://weibo.com/woaitqs)
* Feed订阅：[www.woaitqs.cc/feed.xml](http://www.woaitqs.cc/feed.xml)

------------------------
