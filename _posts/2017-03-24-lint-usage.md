---
layout: post
title: "自定义 Lint 提升代码质量"
description: "自定义 Lint 提升代码质量"
category: "android"
keywords: "lint,custom,JavaScanner"
tags: [android,lint]
---


Android 客户端工程师现在是良莠不齐的阶段，每个人的编程水平不尽相同，这也导致开发出的 APP 在性能和实现效果上面差异很大。Google 也考虑到这个问题，推出了一些常见的编程建议，并通过 lint 这个工具可以告知我们代码中有哪些不合理的实现。Lint 是一款静态代码分析工具，能检查安卓项目的源文件，从而查找潜在的程序错误以及优化提升的方案。

<!--more-->

--------------

### lint 基本使用规则

我们看看下面这张图，是不是非常熟悉呢？

![lint example](http://o8p68x17d.bkt.clouddn.com/lint_example.jpg)

这个用黄颜色标示出来的错误，就是 lint 给出我们的提示。当我们在开发的时候，发现用黄色标记出来的代码块的时候，一定不要忽视，要通过 lint 给出的建议将这些潜在问题修复掉。

默认情况下 lint 是启动的，具体 lint 给我们设置了什么样的规则，可以在 Android Studio -> Editor -> Inspections 中进行查看。点击具体的某一项可以看到关于这条规则的详细描述，同时可以设置警告的级别（通过编辑器的颜色来达到不同的警示效果）和能操作的范围。

![android studio lint](http://o8p68x17d.bkt.clouddn.com/studio-lint.png)

我们可以通过 `./gradlew lint` 来对代码进行分析，或者也可以通过 `Analyze > Inspect Code...` 来进行检查。最终会在界面上，显示各种各样的 lint 错误。

再介绍下 lint 的一个妙用，在 Refactor 这个菜单里面有 `Remove Unused Resources` 按钮，能查看项目中没有使用的资源（colors.xml, drawable, dimens.xml etc.），并一键移除。

在介绍完 Lint 的用处之后，大家就已经可以享受 Lint 给我们带来的好处了。在这个[ http://tools.android.com/tips/lint-checks 链接 ](http://tools.android.com/tips/lint-checks)中展示了目前官方推荐的应该遵循的规则，同样可以在这个[https://android.googlesource.com/platform/tools/base/+/master/lint/libs/lint-checks/src/main/java/com/android/tools/lint/checks](https://android.googlesource.com/platform/tools/base/+/master/lint/libs/lint-checks/src/main/java/com/android/tools/lint/checks) 链接中查看其规则是如何实现的。

### 自定义 lint 规则

在某些情况下，我们需要自己去制定一些规则，然后利用 Android 的 Lint 工具帮助我们自动发现某些问题。例如对低版本 Android 系统的兼容性考虑，需要使用 support 包中的 Fragment，因而可以实现一个自定义的规则，来监控项目中对 Fragment 的使用。

接下来，看看如何来自定义 Lint 规则。

#### 添加 Lint 依赖

为了将这个自定义的规则应用到项目中去，需要新建一个项目，添加相应的规则后，打包生成 aar 或者 jar。

```groovy
configurations {
    all*.exclude group: 'commons-logging', module: 'commons-logging'
    all*.exclude group: 'org.apache.httpcomponents', module: 'httpclient'
}

// lint dependencies
compile 'com.android.tools.lint:lint-api:25.3.0'
compile 'com.android.tools.lint:lint-checks:25.3.0'
```

建议采用 25 版本以上，在 25 版本引入了 JavaPsiScanner，比原来的 JavaScanner 强大很多，而且可以扩展到以后的 Kotlin 的语言上去。

#### 监控代码

Lint 提供了多种维度的监控，从 Java 文件到 XML 文件，从资源文件夹到 Gradle 文件。当需要监控某种文件的时候，只需要实现对应的 `Detector`。

* Detector.XmlScanner

* Detector.JavaPsiScanner

* Detector.ClassScanner

* Detector.BinaryResourceScanner

* Detector.ResourceFolderScanner

* Detector.GradleScanner

* Detector.OtherFileScanner

这里我们以 ToastDetector 为例进行简单的讲解，这个 ToastDetector 完成的功能就是检查是否有 Toast 创建成功但并没有展示的情况。

首先我们需要创建对应的 Issue，Issue 的构建方法定义如下。id 是唯一标示、briefDescription 是简介，explanation 是详细解释，category 是指错误所属的类别（性能、安全、可用性等等），priority 是指优先级，severity 是指 Lint 警告⚠️的级别，最后的 implementation 将 Issue 与这个类关联起来。

```java
@NonNull
public static Issue create(
        @NonNull String id,
        @NonNull String briefDescription,
        @NonNull String explanation,
        @NonNull Category category,
        int priority,
        @NonNull Severity severity,
        @NonNull Implementation implementation) {
    return new Issue(id, briefDescription, explanation, category, priority,
            severity, implementation);
}
```

首先判断方法中是否含有对 Toast 的调用。

```java
if (!context.getEvaluator().isMemberInClass(method, "android.widget.Toast")) {
    return;
}
```

定义一个 ShowFinder，来查找是否有 Toast 没有调用 show 方法。

```java
@Override
public void visitMethodCallExpression(PsiMethodCallExpression node) {
    super.visitMethodCallExpression(node);

    if (node == mTarget) {
        mSeenTarget = true;
    } else {
        PsiReferenceExpression methodExpression = node.getMethodExpression();
        if ((mSeenTarget || methodExpression.getQualifier() == mTarget)
                && "show".equals(methodExpression.getReferenceName())) {
            // TODO: Do more flow analysis to see whether we're really calling show
            // on the right type of object?
            mFound = true;
        }
    }
}
```

最后如果没有找到，通过 report 接口进行上报。

```java
if (!finder.isShowCalled()) {
    context.report(ISSUE, call, context.getLocation(call.getMethodExpression()),
            "Toast created but not shown: did you forget to call `show()` ?");
}
```

--------------

### 文档信息

* 版权声明：自由转载-非商用-非衍生-保持署名（[创意共享3.0许可证](http://creativecommons.org/licenses/by-nc-nd/3.0/deed.zh)）
* 发表日期：2017年3月29日
* 社交媒体：[weibo.com/woaitqs](http://weibo.com/woaitqs)
* Feed订阅：[www.woaitqs.cc/feed.xml](http://www.woaitqs.cc/feed.xml)

------------------------
