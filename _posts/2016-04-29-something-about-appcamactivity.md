---
layout: post
title: "用好AppCompatActivity"
description: "AppCompatActivity的由来，自定义调色板，Toolbar，兼容的退出动画"
keywords: "AppCompatActivity, Toolbar, 调色板, 退出动画"
category: "android"
tags: [android, program, appcompat]
---
{% include JB/setup %}

Android AppCompatActivity 已经出来一段时间，取代了原来的 ActionBarActivity，在使用一段时间的AppCompatActivity，发现也踩过一些坑，这里做一些总结，打算摘抄到我的小本本上面，以后老了，坐着摇椅慢慢看。 ：）

<!--break-->

----------

### AppCompatActivity 的由来

> The AppCompat Support Library started with humble, but important beginnings: a single consistent Action Bar for all API 7 and higher devices. In revision 21, it took on new responsibility: bringing material color palette, widget tinting, Toolbar support, and more to all API 7+ devices. With that, the name ActionBarActivity didn’t really cover the full scope of what it really did.

引用了官方 ChangeLog 里面的一句话，简单翻译如下：Android 官方的 Support Library 从很简陋的情况，发展到现在这种地步。第一个版本的重点在于：对所有API 7 或者更高版本支持 ActionBar。而在 version 21 版本后，将material color支持，调色板，toolbar支持同样适配到API 7 或者更高版本。然而原来的 ActionBarActivity 不足以覆盖这些新特性，因而 AppCompatActivity 就应运而出。

这次改动不仅是一次对 ActionBarActivity的重命名，而是实现了一定程度的重构。将 APPCompatActivity 中的特性抽取成 AppCompatDelegate ，而通过组合 AppCompatDelegate 这个类的方式，可以在普通Activity里面使用 AppCompat 的相关特性。这样的设计是非常优秀的，如果使用继承来实现这样的方式，那么势必会使得结构复杂度升高，而组合就不一样，使得你能在不增加层级的基础之上，拥有更灵活的适配能力。

----------

### AppCompatActivity 做了什么？

#### 自定义调色板

```xml
<resources>
    <!-- Base application theme. -->
    <style name="AppBaseTheme" parent="Theme.AppCompat">

        <!-- customize the color palette -->
        <item name="colorPrimary">@color/material_blue_500</item>
        <item name="colorPrimaryDark">@color/material_blue_700</item>
        <item name="colorAccent">@color/material_green_A200</item>
    </style>
</resources>
```

AppCompat 适配了Activity material的主题，使得你只需要指定3种颜色就可以，完成对主题的自定义。
如下图所示，需要定义的颜色如图，其中具体的对应关系是：

* colorPrimary 对应ActionBar的颜色。
* colorPrimaryDark对应状态栏的颜色
* colorAccent 对应EditText编辑时、RadioButton选中、CheckBox等选中时的颜色。

![几种主要的颜色](http://www.jcodecraeer.com/uploads/20150430/1430371625123828.png)

#### Toolbar的支持

当我们在使用 ActionBar 的时候遇到了很多自定义的需求，但发现 ActionBar 的自定义支持实在太差劲 ：(
Toolbar 用了一劳永逸地解决自定义的问题，直接采用了 ViewGroup，是的，你没有看错，这样的自定义程度也太强了吧。那么 Toolbar 是怎么设置 Icon, Title的呢。

```java
TitleTextView = new TextView(context);
mTitleTextView.setSingleLine();
mTitleTextView.setEllipsize(TextUtils.TruncateAt.END);
```
Title Icon这些都是动态生成，即时插拔的。那么同理，这样也能进行自定义View的绘制，非常的灵活。如果想要自定义 Toolbar，首先需要在 Themes.xml 里面指定不需要系统默认的Toolbar，然后在相应的 layout 文件中添加上自定义的 Toolbar.

```xml
<style name="AppBaseTheme" parent="Theme.AppCompat.Light.NoActionBar">

	<!-- customize the color palette -->
	<item name="colorPrimary">@color/material_blue_500</item>
	<item name="colorPrimaryDark">@color/material_blue_700</item>
	<item name="colorAccent">@color/material_green_A200</item>

</style>
```

而 Toolbar 也可以在 Fragment 中使用，只需要声明`setHasOptionsMenu(true);`，并复写`onCreateOptionsMenu`方法，并在 `onOptionsItemSelected` 中处理相应的点击事件，就能很轻松地使用 Toolbar 的菜单。

----------

### 不可思议的 Activity 退出动画

在 AppCompactActivity 出来后，做 Activity 的切换动画变得非常轻松，一般而言有两种实现方式。

#### 在主题中指定

<item name="android:windowAnimationStyle"></item> AppCompatActivity 里面可以指定Activity切换动画，通过分别指定进入退出的动画即可，非常简单。

```xml
<style name="ActivityInOutAnimation" parent="@android:style/Animation.Activity">
	<item name="android:activityOpenEnterAnimation">@android:anim/slide_in_left</item>
	<item name="android:activityOpenExitAnimation">@android:anim/slide_out_right</item>
	<item name="android:activityCloseEnterAnimation">@android:anim/slide_in_left</item>
	<item name="android:activityCloseExitAnimation">@android:anim/slide_out_right</item>
</style>
```

#### 在代码中指定

在 StartActivity 和 finish 的时候，指定`overridependingtransition` 也可以进行相应的动画，代码非常简单，各位可以自行 Google.

#### 兼容的退出动画

各位在实际的操作过后，可能还是发现指定的退出动画，不能生效？无论哪种实现方式，都不能很好的工作？这是为什么了？在查看源码，和相应的说明后，还是没有发现相关的原因，不过找到了一种解决方案，这里共享出来。

既然正常的退出动画，无法生效，那么就自己做动画。

1. 拿到整个 Layout 的Root View，对这个 View 做相应的退出效果
2. 拦截相应的 BackPressed 事件，同时在这个时候，启动退出动画
3. 在 Animation 结束的时候，结束掉整个 Activity
4. 由于在 动画结束的时候，依然会显示 Window 的背景颜色，造成闪动的异常。这时候需要将 这个 Activity 指定为下图所示的 `AppTheme.Transparent`，由于是透明的，在退出动画后的 Activity 不会被销毁，这样动画结束后，就能立即看到，从而避免了闪动的效果。

```xml
<style name="AppTheme.Transparent">
	<item name="android:windowBackground">@color/transparent</item>
	<item name="android:windowIsTranslucent">true</item>
</style>
```
