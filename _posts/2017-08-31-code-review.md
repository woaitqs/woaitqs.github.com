---
layout: post
title: "如何做 Code Review"
description: "如何做 Code Review"
category: "code"
tags: [code, review]
---
{% include JB/setup %}

最近公司的业务正在发展，参与的人也越来越多。但人数的增加也伴随着风险的提升，良莠不齐的人员配置，会使得代码质量难以控制。这个时候，就需要在代码入库之前，加上一道质检步骤，通过这个质检步骤，能够有限地规避一些风险。

本文以 Git 作为说明，其他 VCS 工具，原理也大致相同。

<!--break-->

实现代码审查，现在大体有两种方式，分别是 类 Gerrit 或者 类 GitHub/Gitlab，这篇文章主要介绍后者。

### 类 Gerrit

Gerrit 是最容易实现代码审查的一种方式，它在真正仓库和开发者之间建立了一个审核区域，所有进入代码仓库的代码，都必须先通过这个审核区域。

```shell
git push origin dev/v1.0
```

这是最普通的提交方式，但在 Gerrit 中就不可以。

```shell
git push origin HEAD:refs/for/dev/v1.0
```

上面是正确的提交方式，注意到两者的区别了吗？多了一个 refs/for，这就是 Gerrit 的审核区域，只有通过审核，才能进入主仓库。这里给了一个内部实现图，有兴趣的同学可以参考，这里就不进行额外介绍了。

![gerrit_workflow](http://o8p68x17d.bkt.clouddn.com/gerrit-workflow.png)

### 类 Gitlab/Github

重点说说这类代码审查如何展开，因为现在公司使用的就是这种方案，而且 Gerrit 的颜值太低，哈哈。

#### 权限分组

首先需要对权限进行分组，每个组的拥有不同的权限，一般情况下分为 owner, master 和 developer。owner 拥有最高的权限，master 次之。我们用的最多的权限组就是 master 和 developer。master 能够在保护分支上直接 push，而 developer 就不能。这样通过权限的划分，就能保证属于 develop 这个组的人，在没有代码审查的情况下不能将代码入库。

#### 保护分支

在 Gitlab/Github 上有保护分支这个概念，当一个分支属于保护分支的时候，developer 是不能直接入代码的。

我们所进行的工作，很多时候，都是版本迭代开发。因而会有分支去跟踪当前版本迭代。有一种比较好的划分方式，是 master / dev/version? / features/version?。 master 保存着当前最稳当的代码版本，而dev/version? 则是当前开发版本的代码，features/version? 是这个版本代码开发feature的仓库。

developer 的权限只能在 features/version? 上进行工作。那么如何合并代码到 dev 分支了？就只能通过提交 Merge Request，审核通过后，master 将代码弄到 dev 分支上去。

#### Merge Request

![merge_request_select_branch](http://o8p68x17d.bkt.clouddn.com/merge_request_select_branch.png)

上图就是提交 Merge Request 的界面。开发者可以选择自己所在的 features 分支，将相应的 commit 提交申请到 dev 分支上。master 会看到如下的界面，该界面显示是否需要合并这个 Request。

![merge request](http://o8p68x17d.bkt.clouddn.com/close_issue_mr.png)

是不是很简单？更多信息可以查看官方文档，还有这篇[博客](https://blog.assembla.com/AssemblaBlog/tabid/12618/bid/87980/Git-Review-and-Merge-like-a-Boss.aspx)



-----------------

### 文档信息

* 版权声明：自由转载-非商用-非衍生-保持署名（[创意共享3.0许可证](http://creativecommons.org/licenses/by-nc-nd/3.0/deed.zh)）
* 发表日期：2017年8月31日
* 社交媒体：[weibo.com/woaitqs](http://weibo.com/woaitqs)
* Feed订阅：[www.woaitqs.cc/feed.xml](http://www.woaitqs.cc/feed.xml)

-----------------