---
layout: post
title: "JsonObject null 的神坑"
description: "JsonObject 对null的神奇判断"
keywords: "android Object null"
category: "java"
tags: [java, program, json]
---
{% include JB/setup %}

现在开源的 Json 序列化方案层出不穷，在性能和使用方面，都取得了很好的效果，比较常用的包括 `Gson`,`FastJson` 等等。然后对于初学者而言，或者不想引入额外框架的情况下，在这些场景下，还是会使用 `JsonObject` 这种基础对象。今天的文章，就是说一说 `JsonObject` 的神坑。

<!--break-->

来看看这个例子，这样的一个 Json 键值对。

```json
{
  "userName":null
}
```

和下面的这个 Json 键值对。

```json
{
  "userName":"null"
}
```

区别就在 value 上面，一个是实际的 null，另一个是字符串 "null"。我们知道这两个区别很大，但本着科学的态度，我们先用 Gson 来做一下测试，看看情况是否是我们想象中的那样。

新建了一个 People 的类，方便通过 Gson 进行反序列化。

```java
public class People implements Serializable {

  private static final long serialVersionUID = -7182026066461954852L;

  public String userName;

}
```

再通过实际的代码，来进行测试。

```java
Gson gson = new Gson();
People p1= gson.fromJson("{\n" +
    "  \"userName\":null\n" +
    "}", People.class);
People p2 = gson.fromJson("{\n" +
    "  \"userName\":\"null\"\n" +
    "}", People.class);
Log.d("tango", String.valueOf(TextUtils.isEmpty(p1.userName)));
Log.d("tango", String.valueOf(TextUtils.isEmpty(p2.userName)));
```

最后输出的结果，分别是 `true` 和 `false`, 符合我们的预期。

接下来看看 JsonObject 的表现如何，代码很简单，如下所示。

```java
try {
  JSONObject jsonObject1 = new JSONObject("{\n" +
      "  \"userName\":null\n" +
      "}");
  Log.d("tango", jsonObject1.getString("userName"));
} catch (JSONException e) {
  e.printStackTrace();
}

try {
  JSONObject jsonObject2 = new JSONObject("{\n" +
      "  \"userName\":\"null\"\n" +
      "}");
  Log.d("tango", jsonObject2.getString("userName"));
} catch (JSONException e) {
  e.printStackTrace();
}
```

这个地方不会崩溃吗？按道理上讲 `jsonObject1.getString("userName")` 这里是 null 呀，可是没有！它真的没有，真的没有。即使是 null，不是字符串null，JsonObject 也能神奇地返回字符串 null, 这真是天坑呀，比地缝都大啊。

![天啊](http://o8p68x17d.bkt.clouddn.com/wow.webp)

在知道原因以后，解决起来也很容易，在 JsonObject 提供了判断的方法，是 JsonObject.isNull。当然，可以的话，尽量不用 JsonObject ，毕竟不但慢，而且代码可读性差。

------------------------

### 文档信息
* 版权声明：自由转载-非商用-非衍生-保持署名（[创意共享3.0许可证](http://creativecommons.org/licenses/by-nc-nd/3.0/deed.zh)）
* 发表日期：2016年9月12日
* 社交媒体：[weibo.com/woaitqs](http://weibo.com/woaitqs)
* Feed订阅：[www.woaitqs.cc/feed.xml](http://www.woaitqs.cc/feed.xml)

------------------------
