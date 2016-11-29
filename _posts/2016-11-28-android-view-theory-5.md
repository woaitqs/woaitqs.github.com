---
layout: post
title: "Android View 全解析(五) -- 实践自定义 View"
description: "手把手教你自定义 View"
keywords: "custom, view"
category: "android"
tags: [view, android]
---
{% include JB/setup %}

在前面的[系列文章](http://www.woaitqs.cc/android/2016/10/10/android-view-theory-1.html)之后,大家已经对 View 绘制中的各个过程熟悉了，这篇文章就根据前面所学的知识，来实际动手做一个自定义 View，这也是整个系列的完结篇了，会尽可能地涵盖自定义 View 中的各个方面，以及可能遇到的各种坑。在经过一番考量后，决定使用一个 PagerIndicator 作为例子进行讲解。在开始后面的文章之前，建议 Clone 下 [https://github.com/woaitqs/FPageIndicator](https://github.com/woaitqs/FPageIndicator)，对照着源码阅读，食用效果更佳。

<!--break-->

最后我们需要实现的效果如图所示：

![样式展示](https://cloud.githubusercontent.com/assets/1680722/20701135/42297c0a-b64c-11e6-8eea-ab706946af90.gif)

------------------------

### 确定自定义的样式

一般情况下，我们需要自定义 View 的时候，都要考量有哪些地方需要自定义的，对于上图的情况，可以认为选中和未选中的颜色是可以自定义的，大圈和小圈的自定义也是可以自定义的，还有诸如间距等等这些也是可以的。为了使开发者在使用的时候，能够在 XML 文件中也能进行自定义，我们需要声明 `styleable` 文件，[官方教程](https://developer.android.com/training/custom-views/create-view.html) 有详细说明如何进行属性定义。

对于上图的例子，可以写如下的`styleable` 文件。建议自定义的属性上，添加上同一的前缀，这样更容易区分开来。

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <declare-styleable name="PageIndicator">

        <attr name="pi_count" format="integer" />
        <attr name="pi_out_radius" format="dimension" />
        <attr name="pi_radius" format="dimension" />
        <attr name="pi_un_focus_color" format="color"/>
        <attr name="pi_focus_color" format="color"/>
        <attr name="pi_padding" format="dimension"/>

    </declare-styleable>
</resources>
```

那么如何在自定义的 View 中，将这些设置进去的属性取出来呢？这时候我们要用到 TypedArray，通过这个能属性从 AttributeSet 取出来，同时也需要注意在用户没有设置自定义属性时的默认值。代码也相对很简单，如下所示。

```java
private void initAttributes(AttributeSet attrs) {
  if (attrs == null) {
    return;
  }
  TypedArray typedArray = getContext().obtainStyledAttributes(attrs, R.styleable.PageIndicator);
  count = typedArray.getInteger(R.styleable.PageIndicator_pi_count, 1);
  outRadius = typedArray.getDimension(R.styleable.PageIndicator_pi_out_radius,
      dpToPx(DEFAULT_OUT_RADIUS_SIZE));
  innerRadius = typedArray.getDimension(R.styleable.PageIndicator_pi_radius,
      dpToPx(DEFAULT_INNER_RADIUS_SIZE));
  unFocusColor = typedArray.getColor(R.styleable.PageIndicator_pi_un_focus_color,
      Color.parseColor(DEFAULT_UN_FOCUS_COLOR));
  focusColor = typedArray.getColor(R.styleable.PageIndicator_pi_focus_color,
      Color.parseColor(DEFAULT_FOCUS_COLOR));
  padding = typedArray.getDimension(R.styleable.PageIndicator_pi_padding, 0F);
  typedArray.recycle();
}
```

------------------------

### 初始化相应属性

在构造函数或者 attachToWindow 的时候，就初始化自定义 View 中可能用到的各种变量，例如 Paint，而不是在使用时，再去构造。因为诸如 onMeasure、onDraw 方法可能会被多次调用，频繁地分配对象，可能会引起内存抖动。内存抖动会在短时间内触发多次 GC 操作，从而引起卡顿。我在这篇文章中，[http://www.woaitqs.cc/android/2016/03/30/in-love-with-android-memory](http://www.woaitqs.cc/android/2016/03/30/in-love-with-android-memory) 说明了这种情况是如何发生的。

对于本文的情况，初始化 Paint 就可以了。

```java
private void initPaint() {
  innerUnFocusPaint.setStyle(Paint.Style.FILL);
  innerUnFocusPaint.setColor(unFocusColor);

  innerFocusPaint.setStyle(Paint.Style.FILL);
  innerFocusPaint.setColor(focusColor);

  outPaint.setStyle(Paint.Style.STROKE);
  outPaint.setColor(focusColor);
  outPaint.setStrokeWidth(2F);
}
```

------------------------

### 重载 onMeasure 方法

onMeasure 方法是让 view 自身测量自己的大小，这是关键步骤。分为两种情况，继承自 ViewGroup 的，还是继承自 View 。如果是 ViewGroup 的话，需要在合适的地方，让子 View 测量后，再得到自身的大小。而如果是 View，则只需管好自己的大小即可。

MeasureSpec 是 onMeasure 方法中参数的类型，它通过位运算的方式，涵盖了模式和大小两个数据，通过 getMode 和 getSize 可以分别得到模式和大小。这是要特别针对模式进行说明，如果模式为 EXACTLY，说明 View 的大小是指定了的，那么就使用传入的指即可。如果是 AT_MOST, 那么就得先计算出预期的大小。预期的大小，是指在不考虑外界的情况下，View 所占据的大小，对于本文的例子，预期的大小就是几个圈的大小加上它们之间的间距。在计算之后，与指定的大小取两者最小的。如果是 UNSPECIFIED 的情况，那说明父类对你的大小没有预期，那就使用预期的大小即可。下面的代码是一个很清晰的说明。

最后一定不要忘记，调用 setDimension 方法哦。

```java
if (widthMode == MeasureSpec.EXACTLY) {
  width = widthSize;
} else if (widthMode == MeasureSpec.AT_MOST) {
  width = Math.min(desiredWidth, widthSize);
} else {
  width = desiredWidth;
}
```

更多知识，参考 [http://www.woaitqs.cc/android/2016/10/18/android-view-theory-2](http://www.woaitqs.cc/android/2016/10/18/android-view-theory-2).

------------------------

### 重载 onDraw 方法
onDraw 方法就更为简单了，只是简单地对 Canvas 提供的 API 进行调用。在调用时，一定要对 Android 的坐标系非常清楚才行，知道哪里是 X 轴，哪里是 Y 轴，left，right，top 和 bottom 是相对于哪里的距离。

![坐标轴](http://o8p68x17d.bkt.clouddn.com/gettop_getright.png)

只要计算好每个绘制 View 的坐标，再调用相应的 API 就能看到效果了！Cheers！下面的代码，就绘制了选中和未选中的几个点。再强调下，这里使用的 Paint 是提前初始化完成的哦。

```java
for (int i = 0; i < count; i++) {
  canvas.drawCircle(
      outRadius - innerRadius + padding * i + innerRadius * (1 + 2 * i),
      height / 2,
      innerRadius,
      i == selectedPos ? innerFocusPaint : innerUnFocusPaint);
}
```

在自定义 onDraw 的时候，设计的宗旨是保持无状态性，也就是说，onDraw 是对每一个时刻状态的投诉，例如本例子，就应该是某个滑动时刻的投射。如果在 onDraw 中进行动画，或者起一些异步线程等等，会使得代码的可读性变差。

------------------------

### View 的更新策略

写完自定义 View 后，还得在用户更改属性后或者其他事件后，相应地让 View 更新下。

> requestLayout.
会触发 ViewRoot 的 performTraversal方法，从而重新执行 Measure、Layout，在特定情况下会触发 Draw 操作。

> invalidate.
会触发 Draw 操作，如果 View 的大小和位置没有发生改变，调用这个方法就足以更新页面了。

在本文的例子里面，当用户滑动 ViewPager 时，View 的大小和位置不会发生改变，因而调用 invalidate 就足够了。

```java
@Override
public void onPageScrolled(int position, float positionOffset, int positionOffsetPixels) {
  drawPosition = position;
  drawPositionOffset = positionOffset;
  invalidate();
}
```

至此，整个系列结束，有兴趣的同学，可以 Clone [https://github.com/woaitqs/FPageIndicator](https://github.com/woaitqs/FPageIndicator)这个项目来进行探讨 ：）。

------------------------

### 文档信息
* 版权声明：自由转载-非商用-非衍生-保持署名（[创意共享3.0许可证](http://creativecommons.org/licenses/by-nc-nd/3.0/deed.zh)）
* 发表日期：2016年11月28日
* 社交媒体：[weibo.com/woaitqs](http://weibo.com/woaitqs)
* Feed订阅：[www.woaitqs.cc/feed.xml](http://www.woaitqs.cc/feed.xml)

------------------------
