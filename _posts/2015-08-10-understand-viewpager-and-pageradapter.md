---
layout: post
title: "understand viewpager and pageradapter"
description: "对Viewpager 和 pageradapter 的实现讲解"
keywords: "ViewPager, Android, pageradapter, 坑, 源码解析"
category: "android"
tags: [android, viewpager]
---
{% include JB/setup %}

ViewPager 作为展示一组页面的容器，在Android上被广泛使用，这边文章将围绕 ViewPager 如何显示页面展开，接口如何设计展开。

### PagerAdapter 的接口设计

ViewPager 是与一组页面进行交互的容器，那么怎么设计交互的接口就成为设计成败的关键。我们会发现 ListView 中使用的「通信接口」是 BaseAdapter， 那么类似地，ViewPager 在设计的时候， 同样采用了 Adapter 的设计模式， 通过 PagerAdapter 来实现交互。

我们要达成的协议应该如下，ViewPager 负责显示页面，刷新页面，处理滑动等逻辑，而 PagerAdapter 负责实现如何渲染界面等具体接口。ViewPager 不直接操作页面，把这一切逻辑都放在 PagerAdapter 里面去，甚至页面复用这些逻辑也交由 PageAdapter 处理。那么我们来看看 PagerAdapter 是如何定义的？

<!--break-->

PagerAdapter 提供了4种最基础的方法需要实现。

```java
public Object instantiateItem(ViewGroup container, int position) {
    return instantiateItem((View) container, position);
}

public void destroyItem(ViewGroup container, int position, Object object) {
    destroyItem((View) container, position, object);
}

public abstract int getCount();

public abstract boolean isViewFromObject(View view, Object object);
```

首先是 instantiateItem 方法，这个方法在指定的位置，和容器上面实例化 Page， 需要注意的是这些操作必须在 `{@link #finishUpdate(ViewGroup)}` 之前完成（这个会在后面解释）。 ViewPager 会在合适的时机调用这个方法来显示页面。

DestroyItem 这个方法与 instantiateItem 类似，用于销毁页面，在实现的时候，我们可以在这个时间来做一些缓存或者回收等一些事情。同样这些事情也必须在 `{@link #finishUpdate(ViewGroup)}` 保证执行结束。

getCount() 返回相应的数目

`isViewFromObject(View view, Object object)` 这个方法可以仔细说一下。首先看看 `instantiateItem` 的返回值，这里是 Object，读者可能有疑问了，为什么不是 Fragment 了？ 虽然在大多数 Android 程序中，ViewPager 都是用来显示一系列的 Fragment ，但在设计的时候，我们就不能这么闭塞地思考问题。根据开闭原则，我们对扩展是开放的，因而我们除了可以显示一系列 Fragment 以外，还可以显示 View， 或者其他别的什么，所以这么返回值限定为 Object。 返回值和 View 并没有什么关系，ViewPager 只是用这个来标记这个 item 的，也就是建立起 item <--> object 之间的映射。在这样的情况下，能够做更多事情。

例如我们要实现一个显示水果的 ViewPager, 分别是 apple / banner / pear / peach / watermelon。起初第一个版本，我们使用 Fragment 来显示这些水果。

```java
public enum Fruit {
	Apple, Banner, Pear, Peach, Watermelon
}
```

```java
public Object instantiateItem(ViewGroup container, int position) {
	// new fragment
	FruitFragment fragment = new FruitFragment();
	fragment.setArgument(fruits.getItem(position));

	fragmentTransaction.add(fragment);
	// return result.
	return fragment.
}

public boolean isViewFromObject(View view, Object object) {
	return ((Fragment)object).getView() == view;
}
```

同样我们也可以通过显示 View 来替代 Fragment 的实现。

```java
public Object instantiateItem(ViewGroup container, int position) {
	// new fragment
	View view = ViewUtils.inflate(R.layout.fruit_item);
	view.setImageResource(fruit.getResId());
	// return result.
	return view.
}

public boolean isViewFromObject(View view, Object object) {
	return ((View)object).getView() == view;
}
```
### ViewPager 是如何与 PagerAdapter 进行沟通的

在前面的叙述中，ViewPager 是与 PagerAdapter 进行交互的，在具体实现中，ViewPager 在 PagerAdapter 里面注入了一个 Observer, 在 `setAdapter(PagerAdapter adapter)` 调用 `mAdapter.registerDataSetObserver(mObserver);`。

```java
private class PagerObserver extends DataSetObserver {
    @Override
    public void onChanged() {
        dataSetChanged();
    }
    @Override
    public void onInvalidated() {
        dataSetChanged();
    }
}
```

当PagerAdapter 中的数据发生变化时，PagerAdapter 调用 `mObservable.notifyChanged();` 来通知 ViewPager 进行相应的处理。ViewPager 会收到相应的回调, 在 `dataSetChanged()` 方法中进行相应的处理。

### FragmentPagerAdapter 与 FragmentStatePagerAdapter

Android System 针对大多数都是基于 Fragment 来进行页面展示的，因此实现了两个扩展类 FragmentPagerAdapter 与 FragmentStatePagerAdapter。 这两个类可以认为是对 PagerAdapter 进行了二次封装，实现了对 Fragment 的复用和管理。

在进行封装后，这两个类都只需要实现两个接口就可以 work 了(实际上，我们需要做的事情要远比这两个接口多)。

```java
/**
 * Return the Fragment associated with a specified position.
 * 返回相应的Fragment
 */
public abstract Fragment getItem(int position);

/**
 * Return the number of views available.
 */
public abstract int getCount();
```
先看看 FragmentPagerAdapter 是怎么实现的。 FragmentPagerAdapter 继承了 PagerAdapter ，实现了大部分方法，主要适用于静态页面和页面不太多的情况。页面一旦被渲染出来，就会被保存到 FragmentManager里面去，当页面重新出现的时候，就重新attach上去，这样效率会比较好。

```java
@Override
public Object instantiateItem(ViewGroup container, int position) {

	// 在instantiate的时候，添加Fragment
	// 在 finishUpdate的时候，commit transaction.
    if (mCurTransaction == null) {
        mCurTransaction = mFragmentManager.beginTransaction();
    }

    final long itemId = getItemId(position);

    // Do we already have this fragment?
    String name = makeFragmentName(container.getId(), itemId);
    // fragment 是否已经存在了
    Fragment fragment = mFragmentManager.findFragmentByTag(name);
    if (fragment != null) {
    	// 如果已经存在，就调用重新 attach 上
    	// 需要注意的地方是，如果Fragment new出来后，在viewpager 没被销毁的时候，Fragment 就不会被释放掉
    	// 当页面不在显示的时候，只是 detach from fragment manager.
    	// 因此当我们在Fragment 查询到对应tag 的 Fragment 存在，就直接 attach 上就好。
        if (DEBUG) Log.v(TAG, "Attaching item #" + itemId + ": f=" + fragment);
        mCurTransaction.attach(fragment);
    } else {
    	// 实例化 Fragment，这个就需要自己实现了
    	// 当Fragment实例化出来后，就添加的 fragment manager 里面去.
        fragment = getItem(position);
        if (DEBUG) Log.v(TAG, "Adding item #" + itemId + ": f=" + fragment);
        mCurTransaction.add(container.getId(), fragment,
                makeFragmentName(container.getId(), itemId));
    }
    if (fragment != mCurrentPrimaryItem) {
        fragment.setMenuVisibility(false);
        fragment.setUserVisibleHint(false);
    }

    return fragment;
}

@Override
public void destroyItem(ViewGroup container, int position, Object object) {
    if (mCurTransaction == null) {
        mCurTransaction = mFragmentManager.beginTransaction();
    }
    // destroy的时候，并不销毁 Fragment，只是从detach掉
    if (DEBUG) Log.v(TAG, "Detaching item #" + getItemId(position) + ": f=" + object
            + " v=" + ((Fragment)object).getView());
    mCurTransaction.detach((Fragment)object);
}

@Override
public void finishUpdate(ViewGroup container) {
    if (mCurTransaction != null) {
        mCurTransaction.commitAllowingStateLoss();
        mCurTransaction = null;
        mFragmentManager.executePendingTransactions();
    }
}

@Override
public boolean isViewFromObject(View view, Object object) {
    return ((Fragment)object).getView() == view;
}
```

FragmentStatePagerAdapter 与 FragmentPagerAdapter 类似，区别在于 FragmentStatePagerAdapter 更适合于对 Fragment 页面变化比较多，或者经常发生变动的情况。

```java
@Override
public Object instantiateItem(ViewGroup container, int position) {
    // If we already have this item instantiated, there is nothing
    // to do.  This can happen when we are restoring the entire pager
    // from its saved state, where the fragment manager has already
    // taken care of restoring the fragments we previously had instantiated.

    // 如果 Fragment 存在，那么直接返回，不用add
    // 因为不可见的Fragment 都会被remove掉

    if (mFragments.size() > position) {
        Fragment f = mFragments.get(position);
        if (f != null) {
            return f;
        }
    }

    if (mCurTransaction == null) {
        mCurTransaction = mFragmentManager.beginTransaction();
    }

    Fragment fragment = getItem(position);
    if (DEBUG) Log.v(TAG, "Adding item #" + position + ": f=" + fragment);
    if (mSavedState.size() > position) {
    	// 获取可能存在的状态
        Fragment.SavedState fss = mSavedState.get(position);
        if (fss != null) {
            fragment.setInitialSavedState(fss);
        }
    }
    while (mFragments.size() <= position) {
        mFragments.add(null);
    }
    fragment.setMenuVisibility(false);
    fragment.setUserVisibleHint(false);
    mFragments.set(position, fragment);
    mCurTransaction.add(container.getId(), fragment);

    return fragment;
}

@Override
public void destroyItem(ViewGroup container, int position, Object object) {
    Fragment fragment = (Fragment)object;

    if (mCurTransaction == null) {
        mCurTransaction = mFragmentManager.beginTransaction();
    }
    if (DEBUG) Log.v(TAG, "Removing item #" + position + ": f=" + object
            + " v=" + ((Fragment)object).getView());
    while (mSavedState.size() <= position) {
        mSavedState.add(null);
    }
    mSavedState.set(position, mFragmentManager.saveFragmentInstanceState(fragment));
    mFragments.set(position, null);
    // 不缓存，直接移除掉，这样可以节省内存
    mCurTransaction.remove(fragment);
}
```

### 使用 PagerAdapter 的正确姿势

#### FragmentPagerAdapter 的使用细节

* 根据前面所说的情况，`getItem()` 并不能被确保调用，因此 `getItem()` 在传递参数的时候，只适合传递一些静态的内容。如果我们在 `getItem()` 方法调用了一些需要动态改变的东西，然后使用 `notifyDataSetChanged()` ，会发现不起作用，就是因为这个缘故。如果需要在生成 Fragment 对象后，将数据集中的一些数据传递给该 Fragment，这部分代码应该放到这个函数的重载里。在我们继承的子类中，重载该函数，并调用 FragmentPagerAdapter.instantiateItem() 取得该函数返回 Fragment 对象，然后，我们该 Fragment 对象中对应的方法，将数据传递过去，然后返回该对象（可参考这里的实现[Fragment 如何与Activity 进行交互](http://developer.android.com/guide/components/fragments.html#CommunicatingWithActivity)）。
* 注意在 getItem() 的时候不能重复调用 SetArguments() 方法，这种数据传递方式只可能用一次，在 Fragment 被添加到 FragmentManager 后，一旦被使用，我们再次调用 setArguments() 将会导致 java.lang.IllegalStateException: Fragment already active 异常。因而可以采用前面提及的方法。
* 当显示的页面发生变化的时候，需要 `getItemPosition()` 进行特殊处理。`getItemPosition()` 方法有两个魔法值。这个方法会在 `ViewPager` 需要调用查看当前页面是否发生改变的时候调用, 默认返回 `POSITION_UNCHANGED` 表示页面没有发生改变，这也是我们常出现bug的地方。返回 `POSITION_NONE` 来表示页面已经不存在，或者我们可以返回新的位置。通常我们可以返回 `POSITION_NONE` 来强制进行页面的刷新。

```java
public static final int POSITION_UNCHANGED = -1;
public static final int POSITION_NONE = -2;
```

#### FragmentStatePagerAdapter 的使用细节

`Fragment.setArguments()`这种只会在新建 Fragment 时执行一次的参数传递代码，可以放在 `getItem()`里面，其余的代码应该放在 `instantiateItem()` 里面去
