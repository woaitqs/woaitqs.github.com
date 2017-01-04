---
layout: post
title: "Groovy 闭包 -- Gradle教程(三)"
description: "Gradle 系列教程"
category: "gradle"
keywords: "gradle, android, 教程"
tags: [android, gradle, build]
---
{% include JB/setup %}

Groovy 闭包是代码块，可以被引用、带参数、作为方法参数传递、作为返回值从方法调用返回。

<!--break-->

### 语法简介

先来看看闭包的语法，如下面的所示，很简单，其中方括号表示其可省略。

```groovy
{ [closureParameters -> ] statements }
```

closureParameters 是可选的参数列表，可以是0到多个，参数是可以有类型的，或者无类型的。如果声明了参数，那么其后面的 `->` 则必须添加，用于区分参数和函数体。

再来看看一些具体的例子。

```groovy
// 编号1
{ item -> item++ }

// 编号2
{ item++ }

// 编号3
{ println it }

// 编号4
{ it -> println it }

// 编号5
{ name -> println name }

// 编号6
{
  String x, int y ->
    println "hey ${x} the value is ${y}"
}

// 编号7
{
  reader ->
    def line = reader.readLine()
    line.trim()
}
```

第一个例子，表明了一个没有类型的参数的自增操作；第二个例子是第一个例子的简写，自由一个参数时可以这样简写；第三个和第四个例子，也是同样的意思；第五个例子，是第四个例子的另一种写法，表明参数的名字可以随意规定；第六个例子，则是多类型参数的写法，其中 `${}` 指引用了具体的值；第七个例子，表明表达式可以有多行。

### 闭包在 Groovy 中的特殊属性

闭包在 Groovy 中实际上是类型为 `groovy.lang.Closure` 的对象，我们来看看下面的代码。

```groovy
def listener = {
  e -> println "Clicked on $e.source"
}

assert listener instanceof Closure
```

所以，Closure 里面的方法都可以调用，例如 `listener.call()`。Closure 对象中，有三类非常重要的属性，分别是 this, owner, delegate。

1. `this` 表示定义闭包的外围类。
2. `owner` 表示定义闭包的直接外围对象，可以是类或者闭包。
3. `delegate` 表示一个用于处理方法调用和属性处理的第三方类。

例如我们申明一个这样的闭包。

```groovy
def cl = { name.toUpperCase()}
```

这时候 cl 并不知道任何信息，name 是什么也不清楚。这时候就可以将 delegate 引入，将具体的逻辑委托给另一个类。

```groovy
class Preson {
  String name
}
def person = new Person(name: 'tom')
cl.delegate = person;

assert cl() == 'TOM'
```

### 实际示例

我们通常在使用 task 时，会在后面使用 doLast 这个函数，doLast 接受一个闭包作为参数。

```groovy
task helloWorld {
        doLast {
                println "Hello, world!"
        }
}
```

实际上 doLast 后面只接受一个参数，也就是闭包，如果你使用 doLast(Closure) 也是可以的。

更多信息参考 [http://groovy-lang.org/closures.html](http://groovy-lang.org/closures.html)


------------------------

### 文档信息
* 版权声明：自由转载-非商用-非衍生-保持署名（[创意共享3.0许可证](http://creativecommons.org/licenses/by-nc-nd/3.0/deed.zh)）
* 发表日期：2017年1月3日
* 社交媒体：[weibo.com/woaitqs](http://weibo.com/woaitqs)
* Feed订阅：[www.woaitqs.cc/feed.xml](http://www.woaitqs.cc/feed.xml)

------------------------
