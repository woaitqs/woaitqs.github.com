---
layout: post
title: "独立博客进阶技巧"
description: "如何使得你的博客能够脱颖而出？"
keywords: "github, pages, blog, 博客, SEO, 百度, jekyll"
category: "blog"
tags: [blog, thinking]
---
{% include JB/setup %}

<div style="border:solid 1.5px #ccc;padding:20px 20px 10px 20px;margin-bottom: 20px;border-radius: 6px;">

          <span style="position: relative;padding: 5px 15px 5px 15px;background-color: white;bottom: 30px;font-size: 16px;font-weight: bolder;">摘要</span>

          <span style="position: relative;display: block;bottom: 15px;">
          本文介绍了一些与使用 Jekyll 搭建博客的进阶技巧，帮助你能了解自定义 Jekyll Pages 的方法。主要介绍了生成摘要和图床相关的方法，读者可以逐类旁通，弄出一些更加 awesome 的功能。
          </span>
</div>

读这篇文章之前，建议看看[手把手教你用github pages搭建博客](http://www.woaitqs.cc/2016/06/08/blog-seo-baidu.html) 这里面有详细的使用 Github Pages 进行博客搭建的步骤。接下来我们看看 Jekyll 的一些进阶技巧，这些技巧可以使得你能进行自定义的优化和改进，以期博客能够更好的表达你的观点，让世界看见你。

<!--break-->

-------------

### 了解 Jekyll

在讲解一些技巧前，先来了解下 Jekyll 是如何工作的。首先 Jekyll 是一个博客工具，它所完成的工作是将具有特定格式的 Markdown 文件转换成特定目录下的 HTML 文件。例如 2016-05-15-blog-seo-baidu.md 经过 Jekyll 系统的编译后，会生成 2016/05/15/blog-seo-baidu.html的文件，而我们实际在网站上访问的结果也就是这个 HTML 文件，而不是对应的的 Markdown 文件。

#### 运行环境
既然可以进行编译，那么就必须依附于相应的系统，这里可供选择的就是 nodejs, python, ruby等等，另一个知名的博客系统 Hexo 正是依赖于 nodejs, 而 Jekyll 依赖于的则是 ruby。如果大家对 ruby 有一定了解的话，一定知道 Gem, Gem 就是 ruby 下的包管理器，类似于 apk-get 或者 mac 下的 homebrew.

Gem 包依赖可以通过在根目录下生成 `Gemfile` 文件，或者 在 `_config.yml` 指定即可。下面是 Gemfile 的示例。

```shell
source "https://rubygems.org"

gem 'jekyll'
gem 'jekyll-paginate'
gem 'jekyll-coffeescript'
gem 'jekyll-feed'
gem 'jekyll-sitemap'
gem 'therubyracer'
gem 'execjs'
```

#### yml 标记语言
为了更好地表征 Jekyll 相应的配置，需要通过某种标记语言来说明，常见的就是 Json、xml 等等。而这里采用的是 yml 标记语言。

yml 使用 `--` 作为表示文档开始，`...`是结束的标志，例如下面的片段表示一个文档。

```yml
---
# A list of tasty fruits
fruits:
    - Apple
    - Orange
    - Strawberry
    - Mango
...
```

任何一个缩进的表格，表示从属关系，`-` 则表示同级别的列表关系。

```yml
# Employee records
-  martin:
    name: Martin D'vloper
    job: Developer
    skills:
      - python
      - perl
      - pascal
-  tabitha:
    name: Tabitha Bitumen
    job: Developer
    skills:
      - lisp
      - fortran
      - erlang
```

那么 Jekyll 有哪些功能？我们又是如何对其进行配置的了？Jekyll 使用了 `_config.yml` 作为其配置文件，也就是说，在执行 Jekyll 编译命令时，Jekyll 会通过 `_config.yml` 来加载相应的配置，并通过这些配置来生成相应的目录和文件。例如我们使用的分页，`jekyll-paginate` 插件，会在项目的目录中生成 `page{num}` 目录，并在其中放置相应的 `index.html` 文件，这样一来，读者可以通过 [woaitqs.cc/page2](http://www.woaitqs.cc/page2/) 访问到第二页的内容。

#### jekyll 总结

总结一下就是，Jekyll 运行在 ruby 上，通过 Gem 进行包管理，并最终生成相应的 HTML 文件，而最终浏览器展示的正是这些 HTML 文件。

### Jekyll 的摘要问题

我使用的是 jekyll-bootstrap 的系列主题，其每一篇文章都是完全显示，这实在是有些坑爹，完全剥夺了读者通过摘要来判断是否要继续阅读文章的权利。那么怎么实现如下图的的功能。

![article_read_more](http://o8p68x17d.bkt.clouddn.com/article_read_more.png)

官方并没有提供对摘要功能的支持，因而可以通过人为干预的方式来实现。Jekyll 在语法上是支持管道的，`{{ post.content  || split:'<!--break-->' | first }}` 的意思是，对 `<!--break-->` 进行分割，取其中的第一部分，如果我们在合适的位置 插入 `<!--break-->` 不就是我们想要的摘要了吗？

```xml
{% for post in paginator.posts %}
  <h1><a href="{{ post.url }}.html">{{ post.title }}</a></h1>
  <div class="content">
    {{ post.content  || split:'<!--break-->' | first }}
    <footer>
      <a class="morebox" href="{{ post.url }}.html">Read More →</a>
      <hr size="6px" color="#4285F4">
    </footer>
  </div>
{% endfor %}
```

因而 我们可以在对应的 markdown 文件中插入 `<!--break-->` 就可以，而`<!--break-->` 本身作为注释，不会影响 markdown 的显示。

### 图床技巧

写博客免不了使用图片，图片使用外链的话，可用性不能保证，万一外链挂了，图片找回成本就会很大。

先说说我采用过的几个方案：

1. [sm.ms](http://sm.ms) 最开始使用的是这个网站，界面简介大方，落落有致。在绝大多数时候，这个网站都能解决我的问题，知道最近出现了一些不寻常的波动，经常导致图片挂掉，因为决定弃用这个网站。

2. [tuchang.org](http://tuchuang.org/) 国内的一个图床网站，还没有发现挂掉的问题，不过界面有些操作不方便，也担心其服务挂掉，因而选择弃用。

3. 最终采用了七牛的解决方案，作为国内老牌的 CDN 服务，受到了一定程度的好评，使用 10 块钱，成为标准用户后，这样可以免费的 10G 流量。目前而言 10G 流量对我来说绰绰有余，如果有一天这流量不够用了，那说明博客的访问人数到了一定的量级，这也是值得开心的事情。

如果使用的是 Mac 系统，那么有超级方便的工具，可以让你去使用 `Dropzone`, 当你拖拽图片的时候，通过七牛的插件，可以一键上传图片。在图片上传完成后，点击弹出的通知，即可将图片外链复制到剪切板中，如下图所示。

![dropzone qiniu](http://o8p68x17d.bkt.clouddn.com/dropzone_qiniu.png)

-------------

### 一点感悟

无意中发现了这篇 [国内搞独立域名个人博客并更新的还有多少人？有前途吗？](https://www.zhihu.com/question/19990156) 文章，心里还是有些唏嘘，不知道在另一个四年后，我是否还是继续坚持写作了？想到这里实在有些诚惶诚恐。

在现在微信公众号、头条号，知乎和微博等等各种自媒体，充斥着我们的生活，在这种情况下，是否还是值得写独立博客了？选择了独立博客，往往就意味着孤独和独立，相较于其他自媒体而言，实在有些冷清，毕竟成名的博客只有屈指可数的那么多。

不过即便在这种情况下，仍然相信独立博客，值得我们继续写下去！

任何事情都贵在坚持！
