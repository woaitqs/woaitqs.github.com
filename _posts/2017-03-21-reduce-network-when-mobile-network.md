---
layout: post
title: "通过 Cache-Control、Last-Modified 和 Etag 优化客户端缓存"
description: "通过 Cache-Control、Last-Modified 和 Etag 优化客户端缓存"
category: "android"
keywords: "android, Cache-Control, Last-Modified, Etag"
tags: [android, network]
---
{% include JB/setup %}

日常开发中，为了不让用户等得着急，或多或少都会用到 Cache。对于一个有着固定链接的图片资源而言，我们可以简单地将其存储起来，以供后续的访问。但对于那些 Uri 一致，但内容可能随时变化的情形而言，就显得有些不太合适了。简单的例子就是微博的 FeedLine，特别是你关注的人很多时，同样的 Uri 在几秒内对应的内容都可能差别很多。另一方面，要将判断 Cache 是否有效的逻辑与 Back End(后端) 结合起来，否则一定在客户端写死，就很难满足动态的需求了。在这边文章里面，不讨论如何去实现一个 Cache 系统，而是就如何与后端结合起来保证 Cache 的有效性来说明。

<!--break-->

既然是与服务端协定，那么我们就看看在 Http 协议中有哪些地方可以支持我们的需求。当我们获取到 Response 的时候，通常还会有一些 Http Headers 相关信息，这些信息里面包括内容长度、类型、缓存指令等等。下面的内容就展示了一个常见的 Response Headers，这些 Headers 要求客户端最多缓存 3600 秒，也给出了一个 `pub1259380237;gz` 的校验值。

<img src="http://o8p68x17d.bkt.clouddn.com/http-cache-control-highlight.png" alt="http headers" title="http headers" width="300" height="300" align="right" vspace="20" hspace="20"/>

```text
HTTP/1.x 200 OK
Transfer-Encoding: chunked
Date: Sat, 28 Nov 2009 04:36:25 GMT
Server: LiteSpeed
Connection: close
X-Powered-By: W3 Total Cache/0.8
Pragma: public
Expires: Sat, 28 Nov 2009 05:36:25 GMT
Etag: "pub1259380237;gz"
Cache-Control: max-age=3600, public
Content-Type: text/html; charset=UTF-8
Last-Modified: Sat, 28 Nov 2009 03:50:37 GMT
X-Pingback: http://net.tutsplus.com/xmlrpc.php
Content-Encoding: gzip
Vary: Accept-Encoding, Cookie, User-Agent
```


对于缓存而言，我们主要用了 Etag，Cache-Control 和 Last-Modified。

------------------

### Cache-Control

Cache-Control 是指缓存指令，这个指令控制`谁`在`什么条件`下可以缓存响应以及可以缓存`多久`。这个协定取代了以前的 `Expires` 指令，在 HTTP/1.1 开始支持，在这么长时间后，我们可以认为 Cache-Control 在正常环境下都是支持的。Cache-Control的格式如下：

```text
Cache-control: must-revalidate
Cache-control: no-cache
Cache-control: no-store
Cache-control: no-transform
Cache-control: public
Cache-control: private
Cache-control: proxy-revalidate
Cache-Control: max-age=<seconds>
Cache-control: s-maxage=<seconds>
```

#### 谁可以缓存

public 与 private 用来指定谁可以缓存。public 是指任何资源都可以被缓存下来，即使某些部分需要 http 验证的情况下，另一方面 private 则是要求针对单一用户进行缓存，其他用户是无法使用这一块缓存的。例如，用户的浏览器可以缓存包含用户私人信息的 HTML 网页，但 CDN 却不能缓存。在 Android 环境中，不用特殊考虑这种情形。

#### 怎么缓存

no-cache，no-store 与 no-transform 来指定怎么缓存。“no-cache”表示必须先与服务器确认返回的响应是否发生了变化，然后才能使用该响应来满足后续对同一网址的请求。因此，如果存在合适的校验值 (ETag)，no-cache 会发起往返通信来验证缓存的响应，但如果资源未发生变化，则可避免下载。相比之下，“no-store”则要简单得多。它直接禁止浏览器以及所有中间缓存存储任何版本的返回响应，例如，包含个人隐私数据或银行业务数据的响应。每次用户请求该资产时，都会向服务器发送请求，并下载完整的响应。如果没有指定这个字段，那么就认为是可以缓存的。

#### 缓存多久

指令指定从请求的时间开始，允许获取的响应被重用的最长时间（单位：秒）。例如，max-age=60 表示可在接下来的 60 秒缓存和重用响应。s-maxage 与 max-age 的区别在于是否对 Proxy 生效。

在这种情况下，服务端就可以通过 Cache-Control 来指定客户端如何进行缓存。更多关于 Cache-Control 的知识，可以看看这个链接。[https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cache-Control](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cache-Control)。

------------------

### If-Modified-Since and Last-Modified

除了服务端发送的 Cache-Control 指令外，可以做的更灵活一些，毕竟不是所有请求都适用于不变的 max-age。If-Modified-Since 和 Last-Modified 这一对就是另一种灵活的解决方案。

![If-Modified-Since and Last-Modified](http://o8p68x17d.bkt.clouddn.com/If-Modified-Since-and-Last-Modified.png)

Last-Modified 是由服务端返回的，用于告知客户端最后一次修改是什么时候。客户端需要记录下来这个值，并在下一次请求的时候，通过 If-Modified-Since 这个字段附上上一次服务端返回的 Last-Modified 的值。在这种情况下，服务端就有了两次时间，在通过比对后，就可以知道在这段时间内，内容是否发生了改变。如果没有发生变化，就会返回 `304 NOT MODIFIED` 这个状态码，而不是通常的 200。反之如果发生了变化，就进行正常的返回。

------------------

### Etag and If-None-Match

还记得前面提交的 Etag 吗？这是从另一种维度上来确定 Cache 是否需要更新。服务端会返回相应的 Etag，这个 Etag 客户端不用关心其具体是怎么实现的，只需要能够记录下这个值就行。服务端是通过对内容进行 hash，或者别的算法来生成这样的 Etag，当客户端请求的时候，只需要去检查两者是否相同，即可知道内容有没有发生变化。返回的方式，与前面 If-Modified-Since and Last-Modified 相同。

![Etag and If-None-Match](http://o8p68x17d.bkt.clouddn.com/Etag-and-If-None-Match.png)

------------------

### 应用

或许，我们在日常开发中，没有用到前文中提及的内容。但实际上，我们常用的网络框架早已经支持了，无论是 Volley 还是 Okhttp 等等。日后当我们的应用到达一定量级后，要记得用上这些技巧，能给我们的服务端节省一大笔流量费用，也能优化客户端的体验，提升日活哦。

```java
private final static int CACHE_SIZE_BYTES = 1024 * 1024 * 2;
public static Retrofit getAdapter(Context context, String baseUrl) {
    OkHttpClient.Builder builder = new OkHttpClient().newBuilder();
    builder.cache(
        new Cache(context.getCacheDir(), CACHE_SIZE_BYTES));
    Retrofit.Builder retrofitBuilder = new Retrofit.Builder();
    retrofitBuilder.baseUrl(baseUrl).client(client);
    return retrofitBuilder.build();
}
```

上面的代码，OkHttpClient 中的 Cache 自动帮我们完成了诸如 Last-Modified、Etags 这个功能。

------------------

### 参考文献

1. [https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/http-caching](https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/http-caching)

2. [https://android.jlelse.eu/reducing-your-networking-footprint-with-okhttp-etags-and-if-modified-since-b598b8dd81a1](https://android.jlelse.eu/reducing-your-networking-footprint-with-okhttp-etags-and-if-modified-since-b598b8dd81a1)

3. [https://en.wikipedia.org/wiki/List_of_HTTP_header_fields](https://en.wikipedia.org/wiki/List_of_HTTP_header_fields)


--------------

### 文档信息

* 版权声明：自由转载-非商用-非衍生-保持署名（[创意共享3.0许可证](http://creativecommons.org/licenses/by-nc-nd/3.0/deed.zh)）
* 发表日期：2017年3月21日
* 社交媒体：[weibo.com/woaitqs](http://weibo.com/woaitqs)
* Feed订阅：[www.woaitqs.cc/feed.xml](http://www.woaitqs.cc/feed.xml)

------------------------
