---
layout: post
title: "手把手教你用github pages搭建博客"
description: "如何使用github pages建立你的网站, 并让百度索引你的网站"
keywords: "github, pages, blog, 博客, SEO, 百度, 索引"
category: "blog"
tags: [blog]
---
{% include JB/setup %}

如果给你40分钟，可以搭建一个如下图所示的网站，你愿意吗？如果你愿意，那我们就开始干！

![[woaitqs.cc](http://www.woaitqs.cc) 博客](http://i1.buimg.com/4fe4cfa7bbcff3b0.png)

<!--break-->

----------

### 背景介绍
搭建博客网站有各种各样的方法，根据不同的需求，又不同的做法。如果你只是想单纯做一个博客，和世界分享你的观点和视角，那么我推荐使用 github pages。

1. github page 学习成本最低，相比其他搭建方式而言，不需要太多的服务器基础。
2. 天然基于版本控制，便于对代码进行迁移。日后若想进行博客的迁移，只需要支持 Jekyll 或者 Hexo 就行。
3. Github Pages 本身就是一个 git 项目，其他人可以 fork 或者参与到你的博客建设中，这本身可以独立也可以互相协作，换而言之，这也是你的 contributions , 如下图所示，当别人进入你的 github 主页时，就能看到你的活跃程度了。

![git contributions](http://i1.buimg.com/2a32ea5db3b9f3d6.png)

Github Pages 需要相应的博客引擎来驱动，主流的有两个 [https://hexo.io/](https://hexo.io/) 和 [https://jekyllrb.com/](https://jekyllrb.com/)。两者的使用频率都比较高，我选择了 Jekyll，因为目前国内的 [coding.net](http://coding.net) （国内的github）只支持 Jekyll , 如果日后 github 被墙 (已经短暂发生过...)，还可以无缝切换到 coding.net 上。

----------

### 需要准备的知识和事物

1. 注册一个 Github 账号，点击[这里](https://github.com/)进行注册。
2. 了解一些 git 版本控制系统 的操作原理，如果不熟悉的话，可以通过 [git-scm](https://git-scm.com/) 来进行学习，这个网站在墙外，不能访问的话，可以参考这个极简版本 [git-guide](http://www.bootcss.com/p/git-guide/)。在学习 git 的基本操作后，要学会使用这几个命令 `git pull` 或者 `git fetch`, `git rebase`，以及`git push`。
3. 购买个人域名（可选），使用 github pages，会给你一个 github
的域名，例如我的用户名是 woaitqs，那么我就有一个 [woaitqs.github.io](http://woaitqs.github.io) 的域名。如果你想要有一个属于自己的域名，推荐大家使用 [GoDaddy](http://godaddy.com/)，这个国外的老牌域名服务商。由于这是墙外的服务提供商，在国内还是存在一些不稳定因素，需要搭配 [DNSPod](http://dnspod.cn/) 使用。当然如果你嫌需要2个地方折腾麻烦，可以和我一样选择用阿里的 [万网](https://wanwang.aliyun.com/) 进行域名注册。

![github logo](https://assets-cdn.github.com/images/modules/logos_page/Octocat.png)

这么萌，你确定不要试试吗？

----------

### 配置 Github

Github 是出名的版本控制仓库，所谓仓库就是存放货物的地方，而 Github 就是就是存放代码的仓库。既然是仓库，那么就得要钥匙，不然别人去我们自己的仓库里面偷东西怎么办？同时我们也得知道怎么往仓库里放东西，在这一小节，就来看怎么配置 Github。

* 安装 [Git 工具](https://git-scm.com/downloads)，点击下载并安装。

* 使用仓库前，标明自己的身份（名字和邮箱）。安装git后，通过命令行打开终端，cmd (windows，需配置环境变量) 或者 Terminal，输入一下命令。
![git terminal](http://i1.buimg.com/7f3b9b9d7e596365.png)

```shell
git config --global user.name "YOUR NAME"
git config --global user.email "YOUR EMAIL ADDRESS"
```

* 生成相应的令牌，本地一份，Github 一份，这样 Github 可以在你使用仓库的时候，进行校验确定你的身份。继续在终端里面输入下面的命令。

```shell
cd ~/.ssh
mkdir key_backup
ssh-keygen -t rsa -C "*your_email@youremail.com*"
```
这样在 .ssh 目录下会有 id_rsa.pub ，使用 `cat id_rsa.pub` 命令，可参看具体内容。

* 将 id_rsa.pub(令牌) 传递到 Github ，通过 “Account Settings” > Click “SSH Keys” > Click “Add SSH key” 路径可以提交，如下图所示
![ass ssh key](http://i1.buimg.com/01a5b6d867f92ce0.png)

* 再测试一下

```shell
ssh -T git@github.com
```
如果是下面的反应：

```shell
The authenticity of host 'github.com (207.97.227.239)' can't be established.
RSA key fingerprint is 16:27:ac:a5:76:28:2d:36:63:1b:56:4d:eb:df:a6:48.
Are you sure you want to continue connecting (yes/no)?
```

不要紧张，输入yes就好，然后会看到：

```shell
Hi <em>username</em>! You have successfully authenticated, but GitHub does not provide shell access.
```

也可以参考 [github 官方教程](https://help.github.com/articles/set-up-git/)

----------

### 博客搭建

**后面的 `USERNAME` 均代表你 Github 的账号，再使用时请自行替换。**

* 进入你的 github 主页，选择 `new repository`，进入到 [新建仓库](https://github.com/new) 的界面。新建一个 USERNAME.github.com 的仓库，这里 `USERNAME` 是你的 Github 的用户名，且只能是你 Github 的账户名。
![新建仓库](http://i1.buimg.com/0a1c38869143441e.png)

* 拉取现有的博客模板进入你的仓库。这里`https://github.com/plusjade/jekyll-bootstrap.git` 也可以是其他博客的仓库地址，在最开始的阶段还是建议选择 `https://github.com/plusjade/jekyll-bootstrap.git` 作为入门的仓库，该仓库提供了几款不错的样式。继续在终端中进行这几个步骤。

```shell
git clone https://github.com/plusjade/jekyll-bootstrap.git USERNAME.github.com
  cd USERNAME.github.com
  git remote set-url origin git@github.com:USERNAME/USERNAME.github.com.git
  git push origin master
```

这里所完成的事情，就是拷贝一份别人的仓库，并作为自己的仓库，便于日后自己进行自定义的修改。

* 再耐心等等几分钟，点击 [http://USERNAME.github.com](http://USERNAME.github.com) 就能看到博客界面了。

![cheers.jpg](http://i1.buimg.com/b6fa5a2c3c61eb7f.png)

Cheers！来，朋友干一杯！

----------

### 第一篇博文

在看到界面之后，现在该看看怎么写第一篇博客了。博客的正常工作，需要相关的博客引擎来支持，这就是前文中提及的 [jekyll](https://jekyllrb.com/) , 在 Github Pages 已经安装完成了，因而 前面所使用的 [http://USERNAME.github.com](http://USERNAME.github.com) 才能看到正常的画面。

那么如果想在本地也看到效果后，再上传 Github 的话，可以在本地安装 Jekyll。Jekyll 的安装依赖于 Ruby，因而安装前需要下载 Ruby，具体的步骤如下。

1. **安装Ruby** : 对于 Windows 用户而言，非常简单，从 [rubyinstaller.org](http://rubyinstaller.org/downloads/) 下载即可。而 Mac 用户，最新版的系统已经自带，无需处理。

2. **安装 Gem** : 安装 Ruby 过后，就可以安装可以在 Ruby 运行的软件，有一个工具可以帮我们管理 Ruby 软件，这就是 Gem，类似于 Mac 上的 Homebrew，Ubuntu 上的 apt-get，或者手机上的 豌豆荚 23333。安装方式也很简单，[rubygems.org](https://rubygems.org/pages/download) 从这个网站上进行下载即可。

3. **安装 Jekyll** : 有了软件管理器后，现在要做的事情就是安装今天的主角 Jekyll ，非常简单，一行话的事情。

```shell
gem install jekyll
```

4. **验证安装** : 执行 `jekyll serve` 命令，然后在浏览器访问 [localhost:4000](http://localhost:4000/index.html) 如果一切正常，你就能看到与 Github Pages 上相同的画面了。

5. **享受书写** ：首先创建一篇文章，通过 rake 命令即可。`rake post title="Hello World"` 这样就创建了一篇 YYYY-MM-DD-title-Hello-world.md 的文章。文章采用的是 markdown 语法进行编写，关于 markdown 语法，这里有一篇不错的文章进行介绍，[点击查看](http://wowubuntu.com/markdown/) 。

6. **本地查看与上传** : 同样通过`jekyll serve`，在浏览器中进行查看，如果没什么问题的话，现在就可以上传到服务端。

```shell
# 将新添加的文件加入索引中
git add .
# 将这次的修改作为一个打包
git commit -am "first blood"
# 拉取远程仓库的代码
git fetch
# 与远程仓库的代码进行比对和合并
git rb
# 提交到远程仓库
git push origin master
```

再耐心地等待一小会，访问这个链接 [http://USERNAME.github.com](http://USERNAME.github.com) 即可查看你提交的内容。

----------

### Markdown 语法简介

markdown 语法十分简单，非常有利于写作，这里做一个简单介绍，熟悉的读者可以跳过这一章节。

\# 表示标题，\## 表示2级标题，同理\####表示4级标题

空行表示新的段落，如果不空行的话，markdown 认为是同一段落

\[A](B) 这样样式表示为链接，A为你想要显示的文字，B为实际的链接

!\[A](B) 这种样式表示图片，A为图片的描述文字，B为图片链接

\* 表示无序列表，1,2,3 表示有序列表

以上就是 Markdown 的基本语法。

-----------

### 配置域名

> 一个小插曲
据说*百度爬虫*爬得太猛烈，已经对很多*Github* 用户造成了可用性的问题了，而禁用*百度爬虫*这一举措可能会一直持续下去。换而言之，Github Pages 无法被百度索引。这也是我放弃使用 [woaitqs.github.io](http://woaitqs.github.io) 作为我域名的缘故。

当然你特有的域名，也是对你自己的一种投资，非常值得。

解决上诉的问题，首先需要买一个域名，因为万网自带云解析，因为就直接在万网买了 [woatiqs.cc](http://www.woaitqs.cc) 的域名。当然也可以使用其他域名服务提供商，建议使用 [GoDaddy](http://godaddy.com/)。

接下来需要告知github，现在你有域名了。在根目录下创建 `CNAME` 的文件，一定要`大写`，在文件中输入你的域名即可。在 Github 上直接操作，或者在本地操作，与提交博客的方式一样上传到 Github 都可以。等上几分钟，就可以通过你的域名进行访问了，点击 [woatiqs.cc](http://www.woaitqs.cc) 就和  [woaitqs.github.io](http://woaitqs.github.io) 同样效果了。

![提交 CNAME](http://i1.buimg.com/c26e0d64cf0e9b74.png)

接下来解决百度无法索引的问题，从问题出发，既然百度无法对 Github Pages 进行爬虫，那么对百度走单独的通道，不就解决这个问题了吗？那么单独针对百度的镜像从哪里来？可以是国内的 Coding.net 也可以是自行搭建的 VPS。方法与配置 Github Pages 类似，就不在赘述，可参考 [github屏蔽百度爬虫的解决办法](http://youthyblog.com/2015/08/04/github%E5%B1%8F%E8%94%BD%E7%99%BE%E5%BA%A6%E7%88%AC%E8%99%AB%E7%9A%84%E8%A7%A3%E5%86%B3%E5%8A%9E%E6%B3%95/)。

现在针对百度和默认，配置相关的域名解析即可，如下图所示。
![解析配置](http://i1.buimg.com/c6cfb15a7812860f.png)

当博客访问量不高的时候，可以试试这个外链工具 [seo 外链生成器](http://tool.lusongsong.com/seo/)

----------

### 其他资源

在生成相关的博客后，如果想进一步自定义，还需要对 分页 / 目录 / 主题 / 摘要 / 图床 这些地方进行小的改进，这里先做简单介绍，如果读者有相应需求，可以在详细说明。

分页：
通过 jekyll-paginate 来实现，可自定义每页文章数量。

目录：
使用 markdown 为kramdown时，可使用 TOC 工具自动完成。

主题：
继承他人的样式，或者自定义修改。

摘要：
我采用了<!--break-->这个注释的方式，如果检查到这个注释，就不再往下显示。

图床：
这里重点推荐 [sm.ms](https://sm.ms/)，谁用谁知道。

最后祝大家博客搭建顺利，玩的开心！
