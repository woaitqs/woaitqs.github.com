---
layout: post
title: "从实际例子出发讲解面向组合子编程"
description: "从实际例子出发讲解面向组合子编程，说明OO 和 CO 的区别"
keywords: "组合子, 编程实践, OO, CO"
category: "program"
tags: [combine, program]
---
{% include JB/setup %}

## 前言

一直以来想要表述下，在这几年的时间里面对 OO(`面向对象`) 和 CO(`面向组合子编程`) 的理解，终于在闲暇时间里，可以好好阐述下这个思想。

关于 OO(`面向对象`) 和 CO(`面向组合子编程`) 的理解，有一个很有意思的描述：

OO：白马是白色的马。
CO：白马非马，是由「白」和「马」组成的。

这个理解在一定程度上是正确的，但是更像是面向组件编程，而离「面向组合子编程」还是有一定的差距。面向组件更像是对「组合与继承」的一种反思，组合子编程则更类似于 「编写一门领域语言」。在接下来的章节里面，我们将为什么要「善用组合」，然后如何将思维进阶到「组合子编程」里面来。

<!--break-->

## 善用组合

我们从实际的例子出发，来说明下组合的好处。设想我们需要开发一个 Android App 用来实现一个绘制图形的 App .刚开始 PM 提出了需求，这个 App 需要支持绘制红色的线条，和蓝色的线条，接到这个需求后，作为学过 OO 的小明，当即做出了用继承来实现这个需求的架构，简单的代码实现如下:

```java

public class Line {
	public abstract void draw() {
		// draw line.
	}
}

public class RedLine extends Line {
	public void draw() {
		// draw line.
		// print red color.
	}
}

public class BuleLine extends Line {
	public void draw() {
		// draw line.
		// print blue color.
	}
}

```

这个 App 成功地上线后，用户广泛地好评，PM 的野心又出现了，既然可以画线，那么也可以画方块。 PM 又提出了一个需求， 需要实现对绘制红色方块和蓝色方块的需求。小明想了一下后，在原有的基础上再继承一下，就可以了吧。于是 AppV2 的代码就诞生了:

```java

public class Rect {
	public abstract void draw() {
		// draw Rect.
	}
}

public class RedRect extends Rect {
	public void draw() {
		// draw Rect.
		// print red color.
	}
}

public class BuleRect extends Rect {
	public void draw() {
		// draw Rect.
		// print blue color.
	}
}

```

这样一共有6个类文件了，小明心想这PM该满足了吧。 在App上线一段时间后，用户觉得体验太好了，又提出了新的反馈，需要加入其他画笔。这个时候 PM 又提出了其他的需求了， 比如绿色的圆，黄色的五角星等等。 小明这下爆炸了， 原有的实现里面完全没法满足后续的需求了，整个项目充斥着类的爆炸，复用程度低下。那么如何来解决这些问题了？答案就是「善用组合」。

```java

public interface Drawable {
	void draw();
}

public class Shape implements Drawable {
	public Shape(String type) {

	}

	public void draw() {
		// draw according to type.
	}
}

public class Color implements Drawable {
	public Color(int rgba) {

	}

	public void draw() {
		// draw color by rgba.
	}
}

public class Paint {
	public static void drawRedRect() {
		Shape rect = new Shape(ShapeType.Rect);
		Color red = new Color("#FF231424");
		canvas.draw(rect, red);
	}
}

```

这样实现后，复用性得以提升，逻辑也更加清晰，在下面的章节里面，我们再来思考如何用组合子的方式来考虑这个问题。

## 组合子的思考方式

再次回到这个例子里面，我们再次回到这个需求里面来，需要实现的是一系列的绘图操作。这个 App 里面，我们可以简单地分为几个模块，Color, Shape, Operation; 这几个模块通过组合的方式，就可以用复用度良好的方式来完成整个 App 了。按理说，如果我们采用组合的方式已经能够实现很好的鲁棒性和复用性了，那么还欠缺一些什么了？欠缺一些更加高阶的抽象！

我们的思路往往都是从自顶向下地来进行，先有了要做什么事情，然后再把事情做拆分，最后拆分成可执行的颗粒度足够细化的模块。这种思考方式在 OO 时代就被广泛采用，做出的项目设计方案也往往能够拥有不错的复用性和扩展性。而组合子编程思考的角度是自底向上的，这样考虑的方式会给我们另一种途径来解读项目设计。(PS 自顶向下是一种很好的方式，应该推崇)

当使用组合子编程的时候，我们考虑更多的是从底部出发，水到渠成地满足了需求，当需求变化的时候，也能很快地得以支持。回到前面这个 绘图 App 的例子里面来, 对于绘图这件事情，我们是有什么元操作呢？ 我们需要如何简单地构建基础工具了，如何把设计好的组件的方式用一致的方式拼接起来？这里会有一些 DSL 的意思在里面。

我们来看看如何定义最基础的逻辑，一定是一个最基础的抽象，设想就是一个 Drawable 抽象。

```java
public interface Drawable {
	public void draw(Canvas canvs) {
	}
}
```

那么我们现在是不是可以写 Pen，Color 这些抽象了呢？其实不用急，我们得让一起可以运作起来。我们来构建一个序列化执行的Drawable, 有了这个SerialDrawable，我们就可以支持起顺序架构这个关键功能了。比如我们可以实现前面提及的 RedRect, BlueLine 等等。

```java
public class SerialDrawable implements Drawable {

	private List<Drawable> drawbles = new ArrayList<>();

	public void draw(Canvas canvas) {
		for (Drawable drawable : drawables) {
			drawable.draw(canvas);
		}
	}

	public void appendDrawable(Drawable drawable) {
		drawables.add(drawable);
	}
}
```
另外同样我们可以实现 NonDrawable 来表示空， FilterDrawable 来便是可过滤的Drawable. 通过这些原子的数据，我们就可以架构出整个世界了。（PS to be continued ...）

更多内容请参考 [ajooo-名著类](http://www.blogjava.net/ajoo/category/6968.html)