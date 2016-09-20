---
layout: post
title: "Webview 开发要点"
description: "Android Webview 开发中坑和解决方法"
keywords: "Android, Webview, 坑, 解决方法"
category: "android"
tags: [android, program, webview]
---
{% include JB/setup %}

### 为什么要用Webview

随着 Android 系统的迭代改进，用户体验得到很大改善，开发难度也相对降低不少。但由于开发成本和不够灵活（除了插件化技术和动态补丁技术以外，新功能都需要发版），因而原生网页在现在的 Android Product 依然占有一定的位置，甚至出现了一些例如 Hybrid 框架。Android Sytem 本身也提供了对原生网页的支持控件，这就是今天的主角 `WebView`。

尽管是官方提供的控件，`WebView` 的使用依然是坑无数，想要获得一个较好的用户体验更是难上加难。在接下来的文章中，将详细说一下个人总结出了的一些经验，能够对大家的开发带来一些好处，就更高兴了。

<!--break-->

### webview 基础设置

如果只是想单纯地显示一个静态页面，那么调用 `WebView` 的 loadUrl 即可，然而这只是刚起步。下面是干活，如何正确地配置 `WebView` 的 settings，这些都是实际经过项目验证的，可放心使用。

* █  JS 支持
webview.getSettings().setJavaScriptEnabled(true); 网页的交互是依赖于 JS 的，通过这个开关，可以正确地开启 H5 网页的交互功能。在使用的时候，可能 lint 会提示警告。开启这个功能后，会引入跨域攻击的隐患。如果确信不会引入问题的话，可以加入 @SuppressLint("SetJavaScriptEnabled") 来避免编译器的提示.

* █  自适应屏幕
严格来讲，对手机屏幕适配是原生网页 CSS 和 JS 的工作，但往往现实很骨感，WebView 也需要对 PC 等宽的网页进行小的视频，不至于显示一塌糊涂。
webSettings.setUseWideViewPort(true);
webSettings.setLoadWithOverviewMode(true);
通过设置这两个值，能够保证对 PC 等宽的页面也能良好显示。

* █  缓存策略
webview.getSettings().setCacheMode(int model);
model 可指定 Default、NORMAL(被弃用)、CACHE_ELSE_NETWORK（优先使用Cache，即使根据TTL 和 Soft TTL Cache已经过期）、LOAD_NO_CACHE（不弃用缓存）、LOAD_CACHE_ONLY（只用Cache，网络都不请求了）。
建议在实际开发的时候，使用LOAD_NO_CACHE，来避免一些缓存相关的bug。

* █  BASE64 图片显示
可能大家没有注意点到，没有 Page 的图片在页面上无法显示，这是怎么回事？这些图片本身不是URL，而是通过 base 64 编码后产生的图片。在 5.0 以后，Webview 默认禁止了存在潜在隐患的功能，但这也导致在 5.0 系统上，无法显示Base64 的图片。在5.0 之后系统上，开启 webview.getSettings().setMixedContentMode(WebSettings.MIXED_CONTENT_ALWAYS_ALLOW); 即可解决问题。

### webview JS与本地交互

这部分内容就比较复杂了，篇幅缘故不做过多展开，有需求的可以看这个文档。[https://www.cigital.com/blog/android-webviews-and-the-javascript-to-java-bridge/](https://www.cigital.com/blog/android-webviews-and-the-javascript-to-java-bridge/)

在代码中注入的方式如下：webview.addJavascriptInterface(jsPlugin, PLUGIN);

### webview 播放视频

可能大家在开始的时候，和我一样认为在Webview中播放视频是非常容易，几乎是不需要客户端做处理的。播放视频是容易，想要处理好细节就很难。大家可以试试在播放一个视频之后，返回上一个界面或者回到主界面的时候，你会神奇地发现居然还有声音在播放！

原因就是 [WebView threads never stop!](http://stackoverflow.com/questions/2040963/webview-threads-never-stop-webviewcorethread-cookiesyncmanager-http0-3)

解决方法也很简单，在相应的生命周期中，对 `WebView` 进行相关的处理。由于	`WebView` 本身占用着 plugins、 DOM 等资源，当它离开屏幕的时候，也应该释放其占有的 CPU 、网络资源。

```java
private void destroyWebView() {
	if (webview != null) {
		webview.removeAllViews();
		webview.destroy();
		webview = null;
	}
}

private void pauseWebView() {
	if (webview == null) {
		return;
	}
	webview.onPause();
}

private void resumeWebView() {
	if (webview == null) {
		return;
	}
	webview.onResume();
}
```

### webview 支持网页回滚

在支持 JS 过后，就可以在网页端实现往后倒退的功能，我们知道正常情况下按下返回键是触发 Activity 的回退操作，根本轮不到 Webview 来响应。因而实现这个功能首要前提就是，链接系统的返回事件，先判断网页是否可回滚，如果可以回滚，那么就拦截掉返回事件。

判断方法也很简单：WebView.canGoBack()

最后问题来了，webview 提供了 goBack 方法，但这种方法有一些问题。初始页面为A，点击某个链接跳转到B,B页面重定向到C页面。当调用webview.goBack()时，页面回退到B，然后接着会重定向回C页面。

解决这个问题，有一种方法是依赖于 JS，通过 JS 的 API 可以实现对历史记录的管控。

```java
webview.loadUrl("javascript:window.history.back();");
```

### WebViewClient 与 WebChromeClient

设计到网络的应用就免不了要和各种事件、异常情况打交道，WebViewClient 和 WebChromeClient 则是 `WebView` 提供给我们的回调程序，便于我们处理各种情况。

一般来说，WebViewClient 是负责处理网页相关的事件和异常，几个重要的回调接口在于

* onPageFinished
* onReceivedError
* onPageStarted
* onReceivedSslError (在这里可以忽略一些证书引起的错误，强制执行)

而 WebChromeClient 则是负责一些 JS 加载和进度相关的东西，几个重要的回调是

* getVideoLoadingProgressView
* onReceivedTitle
* getDefaultVideoPoster （视频加载时的默认图）
* onProgressChanged （加载进度回调）

### 强大的调试工具

好东西总是留在最后，做个 web 开发的人都知道 Chrome 提供了强大的 Debug 工具，能够让我们比较方便地进行调试。那么有可能，对于 WebView 界面也能有用这么强大的调试功能吗？ 答案是肯定以及肯定的。

```java
if (BuildConfig.DEBUG && Build.VERSION.SDK_INT
	>= Build.VERSION_CODES.KITKAT) {
   WebView.setWebContentsDebuggingEnabled(true);
}
```

在 API 19以后，Webview 就可以支持内容调试，方法很简单。

1. 打开相应的 WebView 界面
2. 打开相应的调试开关
3. 在 Chrome 中输入 chrome://inspect
4. 享受这个便捷功能吧，骚年！
