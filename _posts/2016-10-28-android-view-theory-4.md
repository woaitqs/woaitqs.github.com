---
layout: post
title: "Android View 全解析(四) -- onDraw"
description: "android view onDraw 的执行过程"
keywords: "onDraw, view"
category: "android"
tags: [view, window, onDraw]
---
{% include JB/setup %}

在前面介绍了 [onMeasure](http://www.woaitqs.cc/android/2016/10/18/android-view-theory-2.html) 用于确定 view 大小，[onLayout](http://www.woaitqs.cc/android/2016/10/25/android-view-theory-3.html) 用于确定 view 的位置后，最后我们看看三大事件中最后压轴出场的 onDraw，这确定了 view 长什么模样。

onDraw 相对前面两者，涉及到的知识面非常的广。从 Canvas 提供的种类繁多的 API，到 Paint、Path 贝塞尔曲线等等高阶的工具，如果要细讲的话，估计这篇文章就得老长老长了。在这些工具的支持下，圆角按钮、复杂的下拉动画等等都手到擒来。关于这些工具的具体用法，在文末提供一些参考链接，有兴趣的同学可以去学习下。

<!--break-->

接下来就用一些实际的例子，来给没有怎么用过自定义 View 的同学开一扇窗，看看在 onDraw 方法里面可以玩出什么样的花样。这些例子的难度由简入难，学习曲线良好。

------------------------

### 基础操作

onDraw 方法中的参数为 canvas，这个类型为 [Canvas](https://developer.android.com/reference/android/graphics/Canvas.html) 的实例，就是我们主要操作的对象。中文意思为画布，也很形象哈，onDraw 就是在画布上将东西画上去。那么既然是把内容画上去，那么第一件事情就是要有一个画笔。Android 提供了这样的工具，也就是 [Paint](https://developer.android.com/reference/android/graphics/Paint.html) 类。

Paint 提供了一些常用的 API，用于自定义画笔。

| 方法名        | 含义           |
| ------------- |:-------------:|
| setColor     | 设置画笔的颜色 |
| setStrokeWidth      | 画笔的宽度 |
| setStyle | 画笔的类型（实心、空心、描边） |
| setShader | 设置着色器 (渐变等效果) |

接下来绘制一个正菱形。绘制菱形的话，可以直接画上去，但需要考虑角度等等复杂问题，有一个更讨巧的方法是先绘制一个正方形，然后以某个角为原点，旋转45度即可 ：）

```java
Paint paint = new Paint();
paint.setColor(Color.RED);
paint.setStyle(Paint.Style.FILL);
paint.setStrokeWidth(10F);
// 以上代码，最好在构造函数时就完成，不要在 onDraw 方法时调用
// 否则会引起内存抖动。

// 绘制正方形
canvas.drawRect(200, 200, 400, 400, paint);

// 绘制菱形，为了区分，修改下颜色
paint.setColor(Color.BLUE);
// 将坐标原点移动到顶点上
canvas.translate(200, 200);
// 以坐标原点为中心，进行旋转
canvas.rotate(45);
// 绘制菱形
canvas.drawRect(0, 0, 200, 200, paint);

```

![效果如图](http://o8p68x17d.bkt.clouddn.com/draw_rect.png)

------------------------

### 绘制操作

首先我们绘制一个简单的背景图，这也是 Canvas 提供的基础功能，包含有重载的多个 drawBitmap 接口。如果没有什么特殊需求，可以调用 drawBitmap(Bitmap bitmap, Matrix matrix, Paint paint) 方法。简单的示例代码如下，这样图片就出现在画布的左上角了。

```java
Bitmap bgBitmap = BitmapFactory.decodeResource(getResources(), R.drawable.spring);
canvas.drawBitmap(bgBitmap, new Matrix(), new Paint());
```

![绘制图片](http://o8p68x17d.bkt.clouddn.com/draw_bitmap.png)

接下来，我们尝试再画一个三角形，学习如何绘制路径。绘制与路径相关的内容，需要用到 Path 这个工具。Path 有许多高阶的用法，例如贝塞尔曲线，在接下来的章节中将简单地进行叙述，这里先只绘制直线。

```java
Paint paint = new Paint();
paint.setStrokeWidth(10F);
paint.setStyle(Paint.Style.STROKE);
paint.setColor(Color.BLACK);
paint.setStrokeJoin(Paint.Join.ROUND);

Path path = new Path();
path.lineTo(0, 200);
path.lineTo(200, 200);
path.close();

canvas.drawPath(path, paint);
```

![直角三角形](http://o8p68x17d.bkt.clouddn.com/draw_tribble.png)

path 关于直线的操作，主要是两个 moveTo 和 lineTo，moveTo 移动下一次绘制操作开始的起点，例如 moveTo(20,20)，那么下一次操作就从(20,20) 这里开始。而 lineTo 就更加直接一点，直接在设定的起点(默认为(0,0))开始，到指定的结束为止绘制一条直线。

在一些参考博客中, 发现他们在 onDraw 中通过绘制不同的 Path 来达到动画的效果，再通过 handler postMessage 的方式来进行动画步骤时间的分配。但我不是很推荐这样的做法，这种情况下 onDraw 方法承担了太多的责任，动画的逻辑应该和这个区分开来。其实 Android 本身提供的动画机制以外，还有 AnimationDrawable 这个很方便的工具来通过 xml 来实现动画。

现在在上面的背景图片上绘制一个会动的飞鸟，效果如下图所示：

![fly bird](http://o8p68x17d.bkt.clouddn.com/fly_bird.gif)

代码实现主要是三个 XML 文件，其中这里使用了 Vector(矢量图)(有兴趣的同学可以看看这个[链接](https://developer.android.com/studio/write/vector-asset-studio.html))。

```xml
<?xml version="1.0" encoding="utf-8"?>
<vector xmlns:android="http://schemas.android.com/apk/res/android"
        android:width="18dp"
        android:height="9dp"
        android:viewportHeight="9"
        android:viewportWidth="18">
    <path
        android:name="bird"
        android:pathData="M2,4c3,-1 6,-1.5 7,2c1,-3.5 4,-3 7,-2"
        android:strokeColor="#20416b"
        android:strokeLineCap="round"
        android:strokeLineJoin="round"
        android:strokeMiterLimit="1.41421"
        android:strokeWidth="2"/>
</vector>

<?xml version="1.0" encoding="utf-8"?>
<objectAnimator
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:duration="500"
    android:interpolator="@android:anim/accelerate_interpolator"
    android:propertyName="pathData"
    android:repeatCount="infinite"
    android:repeatMode="reverse"
    android:valueFrom="M2,4c3,-1 6,-1.5 7,2c1,-3.5 4,-3 7,-2"
    android:valueTo="M4,7c3,-1 4,-4.5 5,-1c1,-3.5 2,0 5,1"
    android:valueType="pathType"/>

<?xml version="1.0" encoding="utf-8"?>
<animated-vector
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:drawable="@drawable/bird_up_and_down">
    <target
        android:name="bird"
        android:animation="@animator/fly"/>
</animated-vector>
```

------------------------

### 贝塞尔曲线

前面提到 Path 可以表达直线和曲线(实际上直线也是曲线的一种)，曲线千变万化，东绕西拐的，有没有一种方式可以去涵盖所有的平面曲线了？有的，答案就是`贝塞尔曲线`。

> 贝塞尔曲线于1962，由法国工程师皮埃尔·贝塞尔所广泛发表，他运用贝塞尔曲线来为汽车的主体进行设计。贝塞尔曲线最初由Paul de Casteljau于1959年运用de Casteljau演算法开发，以稳定数值的方法求出贝兹曲线。
在计算机图形学中贝赛尔曲线的运用也很广泛，Photoshop中的钢笔效果，Flash5的贝塞尔曲线工具，在软件GUI开发中一般也会提供对应的方法来实现贝赛尔曲线。

发现很难找到除了数学公式以外，对贝塞尔曲线进行精确定义的方式。下面借用下这篇博客中[http://www.html-js.com/article/1628](http://www.html-js.com/article/1628) 的示意图，对贝塞尔曲线进行粗略的讲解。

假设，我们有三个点，现在来绘制通过这三个点确定下来的贝塞尔曲线。

![贝塞尔-step1](http://htmljs.b0.upaiyun.com/uploads/1386078258661-bezier-quadratic-step1.png)

我们随便在 AB 线段上找一个点 D，同时在 BC 上找一点 E，使得 AD/AB = BE/BC，如下图所示

![贝塞尔-step2](http://htmljs.b0.upaiyun.com/uploads/1386078272369-bezier-quadratic-step2.png)

这时候，链接 DE 两点，在这条链接线上在找一点 F，使得 DF/DE = AD/AB = BE/BC

![贝塞尔-step3](http://htmljs.b0.upaiyun.com/uploads/1386078293828-bezier-quadratic-step4.png)

上诉就是每个点的确定过程，如果对 AB 线段上的每一个点都进行同样的操作，那么就能得到许多 F 点，最后将这些 F 点链接起来，得到的就是 ABC 三个点对应的贝塞尔曲线。

![贝塞尔-step4](http://htmljs.b0.upaiyun.com/uploads/1386078307084-bezier-quadratic-end.png)

博文还给出了一个过程的展示动画，非常的用心呀。

![效果展示](http://htmljs.b0.upaiyun.com/uploads/1415845715278-bezier-quadratic-animation.gif)

对于 3阶、4阶或者更高阶的贝塞尔曲线，操作是类似的，不过是需要操作的点更多了而已。接下来，我们通过实际的例子，来演示下如何使用贝塞尔曲线，这次的例子是绘制波浪线。

![波浪](http://o8p68x17d.bkt.clouddn.com/draw_wave.gif)

```java
public class WaveView extends View {

  private static final int STEP = 8;
  private static final int COUNT = 5;
  private static final int WAVE_LENGTH = 200;
  private static final int MAX_OFFSET = 200;
  private int offset = 0;

  public WaveView(Context context) {
    this(context, null);
  }

  public WaveView(Context context, AttributeSet attrs) {
    this(context, attrs, 0);
  }

  public WaveView(Context context, AttributeSet attrs, int defStyleAttr) {
    super(context, attrs, defStyleAttr);
    final ScheduledExecutorService executorService = Executors.newSingleThreadScheduledExecutor();
    executorService.scheduleAtFixedRate(new Runnable() {
      @Override
      public void run() {
        getHandler().post(new Runnable() {
          @Override
          public void run() {
            if (offset > MAX_OFFSET) {
              offset = 0;
            }
            offset += STEP;
            invalidate();
          }
        });
      }
    }, 300, 200, TimeUnit.MILLISECONDS);
  }

  @Override
  protected void onDraw(Canvas canvas) {

    Paint paint = new Paint();
    paint.setStrokeWidth(5F);
    paint.setStyle(Paint.Style.STROKE);
    paint.setColor(Color.BLUE);
    paint.setStrokeJoin(Paint.Join.ROUND);

    Path path = new Path();
    path.moveTo(offset, 200);
    for (int i = 0; i < COUNT; i++) {
      int randomHeight = new Random().nextInt(50) + 50;
      path.cubicTo(
          (i + 1F / 3) * WAVE_LENGTH + offset, 200 - randomHeight,
          (i + 2F / 3) * WAVE_LENGTH + offset, 200 + randomHeight,
          (i + 1) * WAVE_LENGTH + offset, 200);
    }

    canvas.drawPath(path, paint);

    super.onDraw(canvas);
  }
}
```

代码也是非常的简单，首先通过 cubicTo 生成波浪，cubic 函数中的前两个函数分别是两个控制点，我们将两个控制点分别置于水平线上下两个地方，这样就能生成一条波浪线。其次在使得每次波浪都便宜固定值，形成涌动的效果。最后为了一个更好的效果，给波浪振动的高度上加上一个随机数。

贝塞尔曲线能提供的功能原不只这些，更多姿势大家就自行解锁了哦 ：）

------------------------

### 参考文献

1. [http://www.gcssloop.com/customview/CustomViewIndex](http://www.gcssloop.com/customview/CustomViewIndex)
2. [https://developer.android.com/training/custom-views/custom-drawing.html](https://developer.android.com/training/custom-views/custom-drawing.html)
3. [http://jeremie-martinez.com/2016/09/15/train-animations/](http://jeremie-martinez.com/2016/09/15/train-animations/)
4. [https://developer.android.com/guide/topics/graphics/2d-graphics.html](https://developer.android.com/guide/topics/graphics/2d-graphics.html)

------------------------

### 文档信息
* 版权声明：自由转载-非商用-非衍生-保持署名（[创意共享3.0许可证](http://creativecommons.org/licenses/by-nc-nd/3.0/deed.zh)）
* 发表日期：2016年11月3日
* 社交媒体：[weibo.com/woaitqs](http://weibo.com/woaitqs)
* Feed订阅：[www.woaitqs.cc/feed.xml](http://www.woaitqs.cc/feed.xml)

------------------------
