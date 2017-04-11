---
ayout: post
title: "更优雅的方式添加 Header 与 Footer"
description: "更优雅的方式给 RecyclerView 添加 Header 与 Footer"
category: "android"
keywords: "RecyclerView,header,footer"
tags: [android,RecyclerView]
---
{% include JB/setup %}

添加头部和尾部的需求，在 Android 开发中特别常见，那么有什么优雅的方法来完成这个工作了？

<!--break-->

### 0X0000 怎么为优雅的方式

通常我们在添加 Header 与 Footer 的时候，需要在 RecyclerView 的 Adapter 中添加相应的逻辑来实现我们的功能。分别需要在 `getViewType`, `onCreateViewHolder` 等方法中针对 header 与 footer 进行修改。但这样会破坏原有的结构，需要在 adapter 中进行代码修改。进一步说，可能还会破坏代码集成结构，强行加入一层继承层次。

优雅的方式，应该是使用组合的，将对 header 与 footer 的支持能力，通过注入（组合）等方式进行，不需要对原有代码进行修改。

早些年，写过一篇文章，关于如何写出更具有扩展性，更为优雅的实现方式。有兴趣的同学，可以参考下。[http://www.woaitqs.cc/program/2015/07/30/programming-by-combining](http://www.woaitqs.cc/program/2015/07/30/programming-by-combining).

### 0X0001 可插拔的HeaderFooterHelper

<img src="http://o8p68x17d.bkt.clouddn.com/Proxy.jpg" alt="http headers" title="http headers" width="250" height="250" align="right" vspace="20" hspace="20"/>

在某些情况下，一个客户不想或者不能直接引用一个对 象，此时可以通过一个称之为“代理”的第三者来实现 间接引用。代理对象可以在客户端和目标对象之间起到 中介的作用，并且可以通过代理对象去掉客户不能看到 的内容和服务或者添加客户需要的额外服务。

想要实现得比较优雅，就需要借鉴 Proxy Pattern 模式的思想了。抽象出一个代理，代理只负责将与 header、footer 相关的逻辑，其他功能则路由给被代理的对象。

1, No need change anyting with your target adapter

不需要修改Target Adapter

2, Not destroy target adapter position

不会破坏Target adapter中的 position (PS: 不需要 +1 -1)

3, Support dynamic add & remove

支持动态添加移除

5, No dependencies code build order

不依赖RecyclerView设置顺序 (eg: 不需要提前设置LayoutManager)

```java
headerFooterHelper = new HeaderFooterHelper<>(pageAdapter,
    new HeaderFooterHelper.HeaderFooterHolderCreator<RippleViewHolder>() {
      @Override
      public RippleViewHolder generateHolder(View itemView) {
        return new RippleViewHolder(itemView);
      }
    });

recyclerView.setAdapter(headerFooterHelper.getWrapperAdapter());
headerFooterHelper.setFooter(footerView);
```

### 0X0002 源码实现

```java
import java.util.List;

import android.support.v7.widget.RecyclerView;
import android.view.View;
import android.view.ViewGroup;

/**
 * 这个类用于在不破坏原有 Adapter 的基础上，实现添加(删除) header(footer) 的功能。
 *
 * usage:
 *
 * <pre>
 *   <code>
 *     HeaderFooterHelper helper = new HeaderFooterHelper(baseAdapter, creator);
 *     recyclerView.setAdapter(helper.getWrapperAdapter());
 *   </code>
 * </pre>
 *
 * @since 2017/4/7.
 * @author qisen.tqs@alibaba-inc.com.
 */
public class HeaderFooterHelper<T extends RecyclerView.ViewHolder> {

  // TODO. currently only support one header and one footer.
  // TODO. support list of footer views.
  // TODO. support set presenter to header and footer.

  private View headerView;
  private View footerView;

  private WrapperAdapter wrapperAdapter;

  public void setHeader(View header) {
    if (header == null) {
      return;
    }
    this.headerView = header;
    wrapperAdapter.notifyDataSetChanged();
  }

  public void setFooter(View footer) {
    if (footer == null) {
      return;
    }
    this.footerView = footer;
    wrapperAdapter.notifyDataSetChanged();
  }

  public void removeFooter() {
    headerView = null;
    wrapperAdapter.notifyDataSetChanged();
  }

  public void removeHeader() {
    headerView = null;
    wrapperAdapter.notifyDataSetChanged();
  }

  public HeaderFooterHelper(
      RecyclerView.Adapter<T> baseAdapter, HeaderFooterHolderCreator<T> creator) {
    wrapperAdapter = new WrapperAdapter(baseAdapter, creator);
  }

  public WrapperAdapter getWrapperAdapter() {
    return wrapperAdapter;
  }

  public interface HeaderFooterHolderCreator<T> {
    T generateHolder(View itemView);
  }

  /**
   * proxy pageAdapter used to add header and footer.
   */
  private class WrapperAdapter extends RecyclerView.Adapter<T> {

    private static final int VIEW_TYPE_HEADER = 1;
    private static final int VIEW_TYPE_FOOTER = 2;

    /**
     * 给 baseAdapter 提供的一个缓冲区，baseAdapter 的
     * viewType将左移 VIEW_TYPE_BASIC_OFFSET 这么多位，这样避免header(footer)的viewType 与之冲突。
     */
    private static final int VIEW_TYPE_BASIC_OFFSET = 3;

    private final RecyclerView.Adapter<T> baseAdapter;
    private final HeaderFooterHolderCreator<T> creator;

    private WrapperAdapter(RecyclerView.Adapter<T> baseAdapter,
        HeaderFooterHolderCreator<T> creator) {
      this.creator = creator;
      assert baseAdapter != null;
      this.baseAdapter = baseAdapter;
    }

    private boolean isHeader(int position) {
      return headerView != null && position == 0;
    }

    private boolean isFooter(int position) {
      return footerView != null && position == getItemCount() - 1;
    }

    @Override
    public void onBindViewHolder(T holder, int position, List<Object> payloads) {
      baseAdapter.onBindViewHolder(holder, headerView != null ? position - 1 : position,
          payloads);
    }

    @Override
    public int getItemViewType(int position) {
      if (isHeader(position)) {
        // 设置了 headerView 的情况.
        return VIEW_TYPE_HEADER;
      } else if (isFooter(position)) {
        // 设置了 footerView 的情况.
        return VIEW_TYPE_FOOTER;
      }
      return baseAdapter.getItemViewType(headerView != null ? position - 1 : position);
    }

    @Override
    public void setHasStableIds(boolean hasStableIds) {
      baseAdapter.setHasStableIds(hasStableIds);
    }

    @Override
    public long getItemId(int position) {
      if (isHeader(position) || isFooter(position)) {
        return RecyclerView.NO_ID;
      } else {
        return baseAdapter.getItemId(headerView != null ? position - 1 : position);
      }
    }

    @Override
    public void onViewRecycled(T holder) {
      baseAdapter.onViewRecycled(holder);
    }

    @Override
    public boolean onFailedToRecycleView(T holder) {
      return baseAdapter.onFailedToRecycleView(holder);
    }

    @Override
    public void onViewAttachedToWindow(T holder) {
      baseAdapter.onViewAttachedToWindow(holder);
    }

    @Override
    public void onViewDetachedFromWindow(T holder) {
      baseAdapter.onViewDetachedFromWindow(holder);
    }

    @Override
    public void registerAdapterDataObserver(RecyclerView.AdapterDataObserver observer) {
      baseAdapter.registerAdapterDataObserver(observer);
    }

    @Override
    public void unregisterAdapterDataObserver(RecyclerView.AdapterDataObserver observer) {
      baseAdapter.unregisterAdapterDataObserver(observer);
    }

    @Override
    public void onAttachedToRecyclerView(RecyclerView recyclerView) {
      baseAdapter.onAttachedToRecyclerView(recyclerView);
    }

    @Override
    public void onDetachedFromRecyclerView(RecyclerView recyclerView) {
      baseAdapter.onDetachedFromRecyclerView(recyclerView);
    }

    @Override
    public T onCreateViewHolder(ViewGroup parent, int viewType) {
      if (viewType == VIEW_TYPE_HEADER) {
        return creator.generateHolder(headerView);
      } else if (viewType == VIEW_TYPE_FOOTER) {
        return creator.generateHolder(footerView);
      } else {
        return baseAdapter.onCreateViewHolder(parent, viewType);
      }
    }

    @Override
    public void onBindViewHolder(T holder, int position) {
      // not support bind logic to header and footer view.
      if (!isHeader(position) && !isFooter(position)) {
        baseAdapter.onBindViewHolder(holder, headerView != null ? position - 1 : position);
      }
    }

    @Override
    public int getItemCount() {
      return baseAdapter.getItemCount()
          + (headerView != null ? 1 : 0)
          + (footerView != null ? 1 : 0);
    }
  }

}
```

--------------

### 文档信息

* 版权声明：自由转载-非商用-非衍生-保持署名（[创意共享3.0许可证](http://creativecommons.org/licenses/by-nc-nd/3.0/deed.zh)）
* 发表日期：2017年4月12日
* 社交媒体：[weibo.com/woaitqs](http://weibo.com/woaitqs)
* Feed订阅：[www.woaitqs.cc/feed.xml](http://www.woaitqs.cc/feed.xml)

------------------------
