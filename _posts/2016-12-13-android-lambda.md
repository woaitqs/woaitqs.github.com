---
layout: post
title: "Android lambda 入门教程"
description: "android lambda 入门教程"
category: "android"
keywords: "lambda, android"
tags: [lambda, android]
---
{% include JB/setup %}

用20分钟的时间，再来了解下 Lambda 表达式。为什么要学习 Lambda 表达式呢？毕竟现在的 Android 使用的 JDK 版本官方并不支持 Lambda。这里列出了一些需要理由，来说明为什么要学习 Lambda 表达式。

* Lambda 表达式在后续的 Android 版本中必将得到官方支持，一些其他的 Android 开发语言例如 kotlin，未雨绸缪总是好事。
* Java8 实现的 Lambda 表达式，有助于我们更好地理解编程中非常重要的几个概念，尤其是 **闭包**，理解数据和函数的等价性。
* 使用 Lambda 能够简化我们的代码，是很甜的语法糖，让我们能更好地专注于实现逻辑。
* Android N 将官方支持 Lambda！

<!--break-->

------------------------

### 背景介绍

也许你听过 *Execution in the Kingdom of Nouns*, 中文翻译也叫 *名词王国里的死刑*，这是很有意思的一篇文章，找了一个翻译的链接，没有读过的同学可以看看， [名词王国里的死刑（翻译） - Hi!Roy!](http://www.dear-shen.com/2016/03/16/%E5%90%8D%E8%AF%8D%E7%8E%8B%E5%9B%BD%E9%87%8C%E7%9A%84%E6%AD%BB%E5%88%91%EF%BC%88%E7%BF%BB%E8%AF%91%EF%BC%89/)。在编写 Java 程序的时候，我们会使用类似于 ViewHelper，Compartor 等等这样的名词，几乎所有的类都是名词，Java 世界里面名词是一等公民，而动词是二等公民。但在另一些编程范式里面，名词和动词是具有同等地位的 ，代表就是函数式编程。在这些更纯粹的编程语言中，函数同样也是数据，可以像数据一样被传递，被消费，有兴趣的同学可以看看 *高阶函数*。引入 Lambda 是Java 对这种情况的小弥补。

![funny](http://o8p68x17d.bkt.clouddn.com/function_program_beauty.png)

闭包是函数式编程中非常重要的概念，个人对闭包这个概念的理解是，能够访问上下文环境，自包含的函数代码块，可以在代码中被传递和试用。Java 的匿名函数一定上可以实现闭包，但有了 Lambda 表达式后可以实现得更加简洁有力。

久而久之，对 Lambda 的呼声越来越高，因此在 Java8 时，官方就推出了 Lambda 表达式。

------------------------

### 语法简介

形如 (arguments) -> (body) 的，就是 Lambda 表达式。我们来看看一些简单的例子。

```java
(int a, int b) -> {  return a + b; }

() -> System.out.println("Hello World");

(String s) -> { System.out.println(s); }

() -> 42

() -> { return 3.1415 };
```

1. Lambda 表达式可以有0个，1个以及多个参数。
2. 参数的类型可以显示声明，也可以通过类型推导（通过上下文环境推导出）得出。
3. 参数用括号包含，多个参数用逗号分隔。
4. 空括号表示无参数。
5. 如果是单个参数，同时类型可推导时，可以是不用括号包含起来。
6. body 部分，可以有 0 行，或者多行
7. 当 body 部分只有一行，且返回类型匹配时，可以不加大括号。
8. 当 body 部分有多行时，则必须用大括号包起来，也就是代码块。

总结地说，Lambda 表达式在语法上就是匿名函数的一种简写方式。

------------------------

### 函数式接口

也许你会好奇，这些语法糖是怎么实现的？就没有一点点要求吗？我们知道 Runnable 是可以用于函数式表达式的，在查看 JDK 文档时，发现了这样的一条注释。

> This is a functional interface and can therefore be used as the assignment target for a lambda expression or method reference.

注释里面提及了 functional interface，Lambda 表达式规定必须是函数式接口。在 java8 中用 @FunctionalInterface 这个注解来表示是函数式接口，这里注意的是，函数式接口必须只能有申明一个抽象方法，如果有大于一个的情况，那么会发生编译错误。

我们以 Runnable 为例子，看看新的 Lambda 表达式怎么用。

```java
//Old way:
new Thread(new Runnable() {
	@Override
	public void run() {
		System.out.println("Old!");
	}
}).start();

//New way:
new Thread(
	() -> System.out.println("New")
).start();
```

空小括号表示没有参数，而代码块只有一块，return 的是 void，所以也可以去掉大括号。

在 java 7 中，已经是函数式接口的有

> java.lang.Runnable

> java.util.concurrent.Callable

> java.security.PrivilegedAction

> java.util.Comparator

> java.io.FileFilter

> java.beans.PropertyChangeListener

此外 java8 还增加了一个包，用来包含其他常用的函数式接口。

> Predicate<T> -- a boolean-valued property of an object

> Consumer<T> -- an action to be performed on an object

> Function<T,R> -- a function transforming a T to a R

> Supplier<T> -- provide an instance of a T (such as a factory)

> UnaryOperator<T> -- a function from T to T

> BinaryOperator<T> -- a function from (T, T) to T

我们来看下面这个例子，Java 会通过编译器，来推导具体是什么类型，即使是泛型的情况。

```java
Callable<String> c = () -> “done”;
```

------------------------

### 词法作用域

Lambda表达式不会从超类中继承任何变量名，也不会引入一个新的作用域。Lambda表达式基于词法作用域，也就是说lambda表达式函数体里面的变量和它外部环境的变量具有相同的语义。例如，我们在 Lambda 外部声明了一个变量，在 Lambda 内部可以直接使用。大家在使用匿名函数的时候，会发现出于线程安全的目的，外部传入的变量必须为 final 的，而 Lambda 并不要求你也同样声明 final，但是在之后不能修改该值。这个概念被称为变量捕获。

在 Lambda 中的this，同样因为词法作用域的问题，指得是创建 Lambda 表达式所有的地方的 this。

------------------------

### Android 开发中该如何使用 Lambda

由于 Android Rom 使用的 Java Class 版本还是 6，7 的版本，在 Android 中使用 Lambda，需要对 Java 8 做一些桥接处理，以作适配的需要。这里可以使用 gradle-retrolambda 这个库，就能很方便地使用 Lambda 了。

1. 下载并安装 Java 8
2. 在 gradle 文件中，加入以下代码。

```java
buildscript {
  repositories {
     mavenCentral()
  }

  dependencies {
     classpath 'me.tatarka:gradle-retrolambda:3.4.0'
  }
}

// Required because retrolambda is on maven central
repositories {
  mavenCentral()
}

apply plugin: 'com.android.application' //or apply plugin: 'java'
apply plugin: 'me.tatarka.retrolambda'
```

------------------------

### 文档信息
* 版权声明：自由转载-非商用-非衍生-保持署名（[创意共享3.0许可证](http://creativecommons.org/licenses/by-nc-nd/3.0/deed.zh)）
* 发表日期：2016年12月12日
* 社交媒体：[weibo.com/woaitqs](http://weibo.com/woaitqs)
* Feed订阅：[www.woaitqs.cc/feed.xml](http://www.woaitqs.cc/feed.xml)

------------------------
