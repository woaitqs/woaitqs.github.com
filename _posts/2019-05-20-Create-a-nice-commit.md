---
title: Commit 也有规范
date: 2019-05-20 14:28:45
layout: post
description: "这篇不讲代码的规范，而是 Commit 的规范。什么？还有 Commit 的规范？"
category: "git"
tags: [git, commit]
---

这篇不讲代码的规范，而是 Commit 的规范。什么？还有 Commit 的规范？
<!--more-->

## 痛点
不知道大家有没有写过这样的提交? 这个提交的问题显而易见，不知道修复了什么问题，这个bug是否有对应的 Bug Id？也不知道这个 Bug 的修复，是否会造成重大的影响？

![错误示例](https://i.loli.net/2019/05/20/5ce248a7af27515682.png)

又比如下面另一个情形，比如同一个版本上面，进行了重构。但每个人在提交 Commit 的时候，规范不一样。有人分别叫`Refactor`,`重构`,`Change`。这种情况下，就不太方便统计这个版本进行了重构。


## 规范
前面的痛点，关键问题在于没有规范。没有规矩，不成方圆。其实 Git Commit 是可以有固定规范的，通过这些有固定规范的 commit，解析这些固定格式，就能知晓这些提交做了什么事情，甚至可以分类查看。

现在沿用得较多的是 Angular.js 项目所使用的规范。

```html
<type>(<scope>): <subject>
// 空一行
<body>
// 空一行
<footer>
```

### Type 
<type> 是表明 commit 的作用，主要用于分类。

![type类型](https://i.loli.net/2019/05/20/5ce248a7cb83777039.png)

### Scope
<Scope> 用于说明 commit 影响的范围，比如`Performance`,`View`等等。
AngularJs 使用的一些 Scope 如下，`**bazel**`,`**compiler**`,`**core**`。 Scope 的定义结合具体的业务来执行就好。

### Subject
<Subject> 是用于显示的简要说明。例如bug的描述、性能的优化等等。

### Body
对上面subject里内容的展开，在此做更加详尽的描述，内容里应该包含修改动机和修改前后的对比。

### Footer
<Footer> 放置一些 Issue 相关的信息。

## 脚手架
我相信这么普遍的问题，一定有很多脚手架出现。果不其然，我找到了不少脚手架，大家可以根据自己的需要来选择。

### Git Commit Template
Idea 有口福了，啊，呸！是福气哈。直接有这个插件，提交的时候使用这个插件即可。

### Commitizen

Commitizen 是一个命令行用于生成标准的 commit 命令行工具。
[GitHub - commitizen/cz-cli: The commitizen command line utility.](https://github.com/commitizen/cz-cli)

### Standard-Version

基于规范化的提交过后，还能做不少其他事情。
1. Bump:  升机版本号。
2. Changelog：生成 change log。
3. Commit：提交代码入库。
4. Tag：打上相应标签。

Standard-Version 一键帮我们完成这些任务，这个工具目前我在使用中，感觉还可以。贴出 AngularJs 的 ChangeLog，让我们瞻仰下。

![Augular ChangeLog](https://i.loli.net/2019/05/20/5ce248c21220118909.png)

为了防止世界被破坏！为了守护世界的和平！贯彻爱与真实的邪恶！冲鸭！

------------------------

### 文档信息
* 版权声明：自由转载-非商用-非衍生-保持署名（[创意共享3.0许可证](http://creativecommons.org/licenses/by-nc-nd/3.0/deed.zh)）
* 发表日期：2018年5月20日
* 社交媒体：[weibo.com/woaitqs](http://weibo.com/woaitqs)
* Feed订阅：[www.woaitqs.cc/feed.xml](http://www.woaitqs.cc/feed.xml)

------------------------