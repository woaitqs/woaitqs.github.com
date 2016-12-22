---
layout: post
title: "Groovy 基本用法 -- Gradle教程(二)"
description: "Gradle 系列教程"
category: "gradle"
keywords: "gradle, android, 教程"
tags: [android, gradle, build]
---
{% include JB/setup %}

我们在命令行执行 Android 打包的时候，通常会执行这样的命令，`gradle installDebug`。这就是 Gradle 的常见用法，`gradle [option...] [task…]`，这里的 installDebug 就是 Tasks 之一。

默认情况下，Gradle 会在执行 build.gradle 这个gradle 文件，读者可能会疑惑「我的build.gradle里面没有 installDebug」这个东西呀？那是因为你声明了 com.android.application 这个 plugin。如果想要执行不同的 gradle 文件，可以通过 -b 来指定。例如 gradle -b solution.gradle hello(hello 是指一个 task)。所以 gradle 其实就是执行各种 Task 的一个工具呀。：）

那么这些 Task 是怎么定义的了？这就涉及到 Groovy 了，让我们走进这个动态语言，看看里面的花花世界。

<!--break-->

--------

![Learn Gradle](http://o8p68x17d.bkt.clouddn.com/gradle_banner.jpg)

### Hello World

从最经典的 Hello World  说起。

```groovy
println "Hello Groovy"
```

就是这么简单，Groovy 作为动态语言，直接执行相应的命令就行了。Groovy 是 Java 的一种方言，基于Java虚拟机的敏捷动态语言，在 Java 强大语言的特性下，添加了 Python、Ruby 等等其他语言的强大特征。好啦，能不吹🐂吗？

------

### 变量申明

申明下变量

```groovy
def x = 5
println x

x = "I have Changed!"
println x

x = new java.util.Date()
println x
```

解释一下上面的代码，可以看到 x 在申明之后，类型在不断地变化，这里需要解释的是 Groovy 是动态语言，其类型是能够动态改变的。同时上述代码对应的类型都是 Java 中的类型，例如这里的 5 就是 Java 中的 BigDecimal，同时每一句话都没有以分号结尾。

---

### 集合

```groovy
//创建一个空的列表
def technologies = []

// 增加一个元素
technologies.add("Grails")

// 左移添加，返回该列表
technologies << "Groovy"

// 增加多个元素
technologies.addAll(["Gradle","Griffon"])

// 删除一个元素
technologies.remove("Griffon")
```

这里演示了一个 List 的操作，其实 add、remove 都是 Java List 对应的接口，在某种程度上 Groovy 是 JDK 的语法糖。

再来看看 Map 的使用

```groovy
//创建空的映射
def devMap = [:]

//增加值
devMap = ['name':'Roberto','language':'Groovy']
devMap.put('lastName','Perez')

//遍历映射元素
devMap.each { println "$it.key: $it.value" }
devMap.eachWithIndex { it, i -> println "$i: $it"}

//判断映射是否包含某键
assert devMap.containsKey('name')
```

在变量中的 it 是 Groovy 闭包的使用，在后续章节再慢慢介绍。可以看到集合的操作，还是相当方便和简单的。

更多资料参考这个链接， [Learn Groovy in Y Minutes](https://learnxinyminutes.com/docs/zh-cn/groovy-cn/)

---

### Task

光说不练假把式，在简单了解前面的基础知识后，我们写一下简单的 task ，来通过 Gradle 进行调用。

还是从最简单的 Hello world 说起。

```groovy
task hello {
    description "This is the task you've created"
    doLast {
        println "Hello Groovy!"
    }
}
```

用 Gradle 来执行下看看效果，`gradle hello`。

![hello task](http://o8p68x17d.bkt.clouddn.com/gradle_hello.png)

这里面的 description，是指对于任务的描述，当执行 `gradle tasks` 就能看到你写的描述了。

再来看看 task 依赖的例子，例如我们有一个上学的task，和一个起床的task，上学的task是必须要在起床这个task执行完毕后才能执行的，也就是依赖于上学这个 task。

```groovy
task getUp {
    doLast {
        println "Good Morning."
    }
}

task goToSchool {
    dependsOn "getUp"
    doLast {
        println "Go to school, right now!"
    }
}
```

我们来执行下这个 `gradle goToSchool`，这样你就会发现 getUp 这个 task 也被执行了。除了 dependsOn 这个关系以外，还有 mustRunAfter、shouldRunAfter 等等，在这些的支持下，能够很方便地进行 task 之间的依赖管理。

如果对这一系列的课程感兴趣的话，就订阅我吧！下一章节，将介绍 Gradle 中非常重要的内容 —— 闭包。

------------------------

### 文档信息
* 版权声明：自由转载-非商用-非衍生-保持署名（[创意共享3.0许可证](http://creativecommons.org/licenses/by-nc-nd/3.0/deed.zh)）
* 发表日期：2016年12月22日
* 社交媒体：[weibo.com/woaitqs](http://weibo.com/woaitqs)
* Feed订阅：[www.woaitqs.cc/feed.xml](http://www.woaitqs.cc/feed.xml)

------------------------
