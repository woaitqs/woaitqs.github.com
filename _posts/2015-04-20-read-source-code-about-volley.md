---
layout: post
title: "VOLLEY 超详细源码解析"
description: "从源码入手，从设计出发，详细说明 volley 是如何实现的"
keywords : "源码, volley, 详细, 设计"
category: "program"
tags: [volley, android]
---
{% include JB/setup %}

## 为什么需要阅读Volley的源码

Volley是Google在2013年推出的一个网络库，用于解决复杂网络环境下网络请求问题。「Google出品，必属精品」，而且Volley被使用在包括「Google Plus」的一系列Google产品中，久经考验。因此我们通过学习Volley的源代码，可以学得很多Android网络处理方面的知识，同时可以看看Google 在设计Volley体系结构的时候，所使用的技巧。

![在多如箭雨的情形下，Volley是如何帮你搞定一切的](http://img.blog.csdn.net/20130702124537953?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdDEyeDM0NTY=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

<!--break-->

## Volley组件化的设计

设计良好的组件，在实现层面上也一定是组件完备的。通过一些基础组件的拼接，来架构起一些伟大的功能。

对于网络请求而已，相应大多数人，在设计之初开始，会想到外界输入一个网络请求，通过回调的方式给调用相应的反馈就可以了，遵循这样的设计思路下去，在顶向下地设计就会有一个架构设计，但如果出现一些需求的变动，这样的架构能否在较小代价的情况下来满足需求的变动呢？另一方面，如果我们试着自底向上地展开设计了？

开始先想想对于网络库而已，我们需要什么样的组件。「网络」（负责通过URL和参数来从网络中请求数据），「缓存」（将数据缓存下来，并提供接口供外界请求），「请求」（对请求的封装，比如参数，方法，优先级等），「响应」（对结果的封装），「错误」（请求过程中发生的问题）。这些Volley提出的实现是「Network.java」,「Cache.java」,「Request.java」,「Response.java」，「VolleyError.java」,现在有了这些基础组件后，还剩下2个工作：

1. 为这些定义的组件提供实现。
2. 尽可能地为这些组件提供一个统一的入口. 「[外观模式](http://zh.wikipedia.org/wiki/%E5%A4%96%E8%A7%80%E6%A8%A1%E5%BC%8F)」

在下面的章节里面，来逐个分析每个组件，最后看看Volley是如何把这些组件联系在一起的。

## Compoment [NetWork]


```java
public interface Network {
    /**
     * Performs the specified request.
     * @param request Request to process
     * @return A {@link NetworkResponse} with data and caching metadata; will never be null
     * @throws VolleyError on errors
     */
    public NetworkResponse performRequest(Request<?> request) throws VolleyError;
}
```

这里的结构十分明晰，从接收Request输入到提供NetworkResponse输出，在发生异常的时候，抛出VolleyError。这里是对网络请求进行的封装，这与普通的网络请求不一样，因而Volley提供了另一个组件**HttpStack**

```java
public interface HttpStack {
    public HttpResponse performRequest(Request<?> request, Map<String, String> additionalHeaders)
        throws IOException, AuthFailureError;
}
```

可以看出来，Network是针对HttpStack进行的包装，Volley实现了2种HttpStack，分别针对2.3以下和4.0以上系统。在2.3系统的时候，还没有UrlConnection, 因此用HttpClient来代替。原因可以参考[这里](http://android-developers.blogspot.hk/2011/09/androids-http-clients.html)

```java
if (stack == null) {
    if (Build.VERSION.SDK_INT >= 9) {
        stack = new HurlStack();
    } else {
        // Prior to Gingerbread, HttpUrlConnection was unreliable.
        // See: http://android-developers.blogspot.com/2011/09/androids-http-clients.html
        stack = new HttpClientStack(AndroidHttpClient.newInstance(userAgent));
    }
}
```

无论是HttpClientStack还是HUrlStack都是对request.getMethod()里面的几乎所有方法，包括PUT，POST，DELETE，GET四种RestFul风格的。这里指出这个，是想提醒调用者除了Get和POST外，还有其他方法可以调用，同时[RestFul](http://en.wikipedia.org/wiki/Representational_state_transfer)风格的API正得到广泛认可。Restful 风格把一切都当做资源，提供增，删，改，查四种方式，Volley是对Restful风格进行了良好的支持。调用方可以通过对Volley进行封装可以实现一个RestFul的RPC client。

再来看看Network的实现**BasicNetwork**，里面有许多值得学习的地方：

1. 对etag，和 lastModified 的支持。当我们把一个Response缓存下来的时候，服务端可能返回Etag和lastModified，服务端通过这两个值就可以判断这个请求是否可以命中服务端的缓存，从而可以加快返回的速度。

```java
if (entry.etag != null) {
    headers.put("If-None-Match", entry.etag);
}

if (entry.lastModified > 0) {
    Date refTime = new Date(entry.lastModified);
    headers.put("If-Modified-Since", DateUtils.formatDate(refTime));
}
```

2. 处理 NOT_MODIFIED 的情况。当服务端返回304的时候，即表示命中了缓存，在这里就不需要再走返回Response的步骤了，直接使用Cache中的数据就可以了。在实现上面，是通过Mock的一个NetworkResponse来实现的。

```java

// Handle cache validation.
if (statusCode == HttpStatus.SC_NOT_MODIFIED) {

    Entry entry = request.getCacheEntry();
    if (entry == null) {
        return new NetworkResponse(HttpStatus.SC_NOT_MODIFIED, null,
                responseHeaders, true,
                SystemClock.elapsedRealtime() - requestStart);
    }

    // A HTTP 304 response does not have all header fields. We
    // have to use the header fields from the cache entry plus
    // the new ones from the response.
    // http://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html#sec10.3.5
    entry.responseHeaders.putAll(responseHeaders);
    return new NetworkResponse(HttpStatus.SC_NOT_MODIFIED, entry.data,
            entry.responseHeaders, true,
            SystemClock.elapsedRealtime() - requestStart);
}

```

3. 获取 Response Contents 的数据。如果responseContents使用了3态，亦即通过null，空和有数据来表示三种状态，这是一种很有意思的编程技巧。重点在于**entityToBytes**方法。 这个方法里面使用了一个字节池，来避免我们每次allocate 一个字节数组的开销。

```java
// Some responses such as 204s do not have content.  We must check.
if (httpResponse.getEntity() != null) {
  responseContents = entityToBytes(httpResponse.getEntity());
} else {
  // Add 0 byte response as a way of honestly representing a
  // no-content request.
  responseContents = new byte[0];
}
```

再来看看ByteArrayPool是如何实现的，其实原理很简单，就是用空间来换取性能，避免OOM。在实现上面是使用了「惰性添加」的方式，最大限度的避免在不调用Volley的时候的开销，但也可以根据实际需求，先new出来一个字节数组。

核心方法是如下2个，分别是 **getBuf** 和 **returnBuf**

```java
public synchronized byte[] getBuf(int len) {
	// 遍历已经分配好了的buffer，然后返回这个，并从数组里面溢出点。
    for (int i = 0; i < mBuffersBySize.size(); i++) {
        byte[] buf = mBuffersBySize.get(i);
        if (buf.length >= len) {
            mCurrentSize -= buf.length;
            mBuffersBySize.remove(i);
            mBuffersByLastUse.remove(buf);
            return buf;
        }
    }
    // 否则新建一个
    return new byte[len];
}
```

```java
public synchronized void returnBuf(byte[] buf) {
    if (buf == null || buf.length > mSizeLimit) {
        return;
    }
    // 持有引用，避免被内存回收
    mBuffersByLastUse.add(buf);
    // 二分搜索
    int pos = Collections.binarySearch(mBuffersBySize, buf, BUF_COMPARATOR);
    if (pos < 0) {
        pos = -pos - 1;
    }
    mBuffersBySize.add(pos, buf);
    mCurrentSize += buf.length;
    // 遍历一下，去掉超出maxSize后最近使用的一个字节数组
    trim();
}
```

## Compoment [Cache]

现在看看Volley是如何实现Cache的，后面再接着分析其他几个组件。Cache里面的核心组件是Entry，这个类封装了Cache的一些细节知识，简单看看。

```java
public static class Entry {
    /** The data returned from cache. */
    public byte[] data;

    /** ETag for cache coherency. */
    // etag 和 lastModified 用来向服务器端请求时带上，便于服务器看是否有缓存。
    public String etag;

    /** Date of this response as reported by the server. */
    // 记录这些数据是什么时候从服务器端返回的，从而可以有相对比较复杂的策略。
    public long serverDate;

    /** The last modified date for the requested object. */
    public long lastModified;

    /** TTL for this record. */
    // Time to Leave 客户端可以通过这个数据来判断当前内容是否过期。
    public long ttl;

    /** Soft TTL for this record. */
    // 在这个时间内，是否需要刷新
    public long softTtl;

    /** Immutable response headers as received from server; must be non-null. */
    public Map<String, String> responseHeaders = Collections.emptyMap();

    /** True if the entry is expired. */
    public boolean isExpired() {
        return this.ttl < System.currentTimeMillis();
    }

    /** True if a refresh is needed from the original data source. */
    public boolean refreshNeeded() {
        return this.softTtl < System.currentTimeMillis();
    }
}
```

在Cache提供的接口里面，除了put和get方法外，还有一个方法是**initialize** 方法，这个在CacheDispatcher启动的时候，会去执行。这里相当于给各种Cache策略，提供了一种在初始化的时候的Hook，通过这个Hook可以实现复杂的策略。

Volley提供了2种cache方式，一是**NoCache**，而是**DiskBasedCache**，NoCache比较简单，重点看看**DiskBasedCache**。
DiskBasedCache 提供了 get 和 remove 方法，便于写入和读取。文件级别的Cache默认最大大小为5M，在初始化的时候，会把内存中的Cache读入到内存中，通过LinkedHashMap来实现。

```java
/** Magic number for current version of cache file format. */
// 这里有一个魔法数，当在读取值的时候，会去判断是否是与这个魔法值相同，以判断是否为Cache.
private static final int CACHE_MAGIC = 0x20150306;
```

## Compoment [Request]

Request 相当于整个Volley系统的输入，通过Request，建立起与内部系统沟通的桥梁，因此**Request**的设计至关重要。我们来分析下，Request是如何与内部系统建立沟通的。

1. 如何让Volley知道，我需要什么样的数据。

```java
@Override
abstract protected Response<T> parseNetworkResponse(NetworkResponse response);
```
首先Request是支持泛型的，通过这个泛型来定义Request需要什么样的数据，同时volley 提供了这样的方法，来指定调用方如何通过response 来变成自己想要的数据。

2. 如何让系统知道我需要优先执行我的任务。Volley通过 Priority 这个来判断优先级，在实际执行里面，是通过 PriorityBlockingQueue 来实现优先级高的队列来执行。

```java
public enum Priority {
    LOW,
    NORMAL,
    HIGH,
    // 目前用于清除Cache
    IMMEDIATE
}
```

3. 如何让 Volley 知道我要怎么去执行任务

这个是最根本的需求，调用方可以通过url，method等内容与Volley进行交互，传递参数通过getParams()来实现的。这样实现的一个好处在于可以把一些校验逻辑放在子类里面，子类里面可以校验参数是否合法，如果不合法，则抛出AuthFailureError异常。

```java
/**
 * Returns a Map of parameters to be used for a POST or PUT request.  Can throw
 * {@link AuthFailureError} as authentication may be required to provide these values.
 *
 * <p>Note that you can directly override {@link #getBody()} for custom data.</p>
 *
 * @throws AuthFailureError in the event of auth failure
 */
protected Map<String, String> getParams() throws AuthFailureError {
    return null;
}
```

4. 如何让 Volley 知道我想进行一些操作

目前volley仅支持 Cancel 操作，当用户想取消某个 request 的时候，实际上是设置了一个标志位，Volley 通过这个标志位来进行判断，以决定后续的操作。

## Compoment [Request 和 Error]

Response 和 Error 就是Volley 对调用方的输出，在经历对调用方屏蔽内部细节过后，将Response 告诉调用者。Error 就是对 Exception 的简单封装，这里也就不细细描述了。

```java
/** Callback interface for delivering parsed responses. */
public interface Listener<T> {
    /** Called when a response is received. */
    public void onResponse(T response);
}

/** Callback interface for delivering error responses. */
public interface ErrorListener {
    /**
     * Callback method that an error has been occurred with the
     * provided error code and optional user-readable message.
     */
    public void onErrorResponse(VolleyError error);
}
```

## Volley 是如何整理各个组件的

#### Volley 的体型结构

![Volley 体系结构](http://img.blog.csdn.net/20130702124824156?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdDEyeDM0NTY=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

在Volley 的结构里面，组件都已经定义完毕了，剩下的工作就是如何把这些组件以合适的方式组合起来，供调用者使用。

#### RequestQueue的外观模式

RequestQueue 封装了 Request 队列的一系列操作，理论上用户知道RequestQueue就足够了，通过这个队列来进行任务的添加和取消，当Request 结束的时候，给调用者相应的回调即可。

```java
public void start() {
 	   // 这里无需去管是否马上stop成功，因为线程引用已经被修改了
        stop();  // Make sure any currently running dispatchers are stopped.
        // Create the cache dispatcher and start it.
        mCacheDispatcher = new CacheDispatcher(mCacheQueue, mNetworkQueue, mCache, mDelivery);
        // 启动Cache dispatcher
        mCacheDispatcher.start();
        // Create network dispatchers (and corresponding threads) up to the pool size.
        // 启动network dispatcher
	    for (int i = 0; i < mDispatchers.length; i++) {
        NetworkDispatcher networkDispatcher = new NetworkDispatcher(mNetworkQueue, mNetwork,
                mCache, mDelivery);
        mDispatchers[i] = networkDispatcher;
        networkDispatcher.start();
    }
}
```

#### CacheDispatcher & NetworkDispatcher 的操作

仔细想想，Volley 其实是一个生产者和消费者系统，调用方是生产者，而Volley是消费者。调用方通过RequestQueue 生产Request，而Vollery 消费Request 从而得到Response。那么负责调配这些生产者和消费者的就是Dispatcher，分别是Cache 和 Network 的Dispatcher。

Dispatcher 在实现上，其实比较简单。首先Dispatcher是Thread，线程的Run方法里面，是一个While循环，Run方法在开始的时候，会去读取Request，读取不到会一直Block在哪里；在读取完成后，就开始走相应的逻辑，比如写入缓存或者从网络中读取数据。

```java
@Override
public void run() {
    if (DEBUG) VolleyLog.v("start new dispatcher");
    Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);

    // Make a blocking call to initialize the cache.
    mCache.initialize();

    while (true) {
        try {
            // Get a request from the cache triage queue, blocking until
            // at least one is available.
            final Request<?> request = mCacheQueue.take();
            request.addMarker("cache-queue-take");

            // If the request has been canceled, don't bother dispatching it.
            if (request.isCanceled()) {
                request.finish("cache-discard-canceled");
                continue;
            }

            // Attempt to retrieve this item from cache.
            Cache.Entry entry = mCache.get(request.getCacheKey());
            if (entry == null) {
                request.addMarker("cache-miss");
                // Cache miss; send off to the network dispatcher.
                mNetworkQueue.put(request);
                continue;
            }

            // If it is completely expired, just send it to the network.
            if (entry.isExpired()) {
                request.addMarker("cache-hit-expired");
                request.setCacheEntry(entry);
                mNetworkQueue.put(request);
                continue;
            }

            // We have a cache hit; parse its data for delivery back to the request.
            request.addMarker("cache-hit");
            Response<?> response = request.parseNetworkResponse(
                    new NetworkResponse(entry.data, entry.responseHeaders));
            request.addMarker("cache-hit-parsed");

            if (!entry.refreshNeeded()) {
                // Completely unexpired cache hit. Just deliver the response.
                mDelivery.postResponse(request, response);
            } else {
                // Soft-expired cache hit. We can deliver the cached response,
                // but we need to also send the request to the network for
                // refreshing.
                request.addMarker("cache-hit-refresh-needed");
                request.setCacheEntry(entry);

                // Mark the response as intermediate.
                response.intermediate = true;

                // Post the intermediate response back to the user and have
                // the delivery then forward the request along to the network.
                mDelivery.postResponse(request, response, new Runnable() {
                    @Override
                    public void run() {
                        try {
                            mNetworkQueue.put(request);
                        } catch (InterruptedException e) {
                            // Not much we can do about this.
                        }
                    }
                });
            }

        } catch (InterruptedException e) {
            // We may have been interrupted because it was time to quit.
            if (mQuit) {
                return;
            }
            continue;
        }
    }
}
```