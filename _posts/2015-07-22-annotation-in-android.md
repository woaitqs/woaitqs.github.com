---
layout: post
title: "Annotation in Android"
description: ""
category: ""
tags: [Annotation, Android]
---
{% include JB/setup %}

## 什么是注解

Java 注解又成为 Java 标注， 是 Java 语言在5.0版本以上开始支持加入源代码的特殊语法元数据。

我们常见的 @Override， @Despited 等这些都是注解，注解是一种可以在 Runtime， Source， Class 三个级别上来提供给我们进行语义表达的特殊数据格式。

JDK 定义了四中基础的注解元数据，分别是 @Documented, @Retention, @Target, @Inherited。

1. @Documented 用于描述其它类型的annotation应该被作为被标注的程序成员的公共API
2. @Retention 定义了该Annotation被保留的时间长短：某些Annotation仅出现在源代码中，而被编译器丢弃；而另一些却被编译在class文件中；编译在class文件中的Annotation可能会被虚拟机忽略，而另一些在class被装载时将被读取（请注意并不影响class的执行，因为Annotation与class在使用上是被分离的）。
3. @Inherited 是否可以继承
4. @Target Annotation的作用域

```java
public enum ElementType {
    /**
     * Class, interface or enum declaration.
     */
    TYPE,
    /**
     * Field declaration.
     */
    FIELD,
    /**
     * Method declaration.
     */
    METHOD,
    /**
     * Parameter declaration.
     */
    PARAMETER,
    /**
     * Constructor declaration.
     */
    CONSTRUCTOR,
    /**
     * Local variable declaration.
     */
    LOCAL_VARIABLE,
    /**
     * Annotation type declaration.
     */
    ANNOTATION_TYPE,
    /**
     * Package declaration.
     */
    PACKAGE
}
```

```java
@Documented // 在文档中可描述
@Retention(RetentionPolicy.RUNTIME) // 在运行时保留
@Target(ElementType.ANNOTATION_TYPE) // 作用在ANNOTATION上面
public @interface Target {
    ElementType[] value();
}
```

## 如何用注解来构建强大的功能了？

原来我们写的代码作用域都在 「Java Source Code」中，而如今 注解提供了一个机会，使得我们用更优雅，语义更明显的方式来完成更高层面的复用。举个例子，在进行网络请求的时候，我们需要指定网络获取的方法(GET / POST / DELETE ...)，以及相应的URL和参数。

```java
public Data request(Method method, String url) {
    // handle request
}
```

```java
@Method(HttpRequest.GET)
@URL("http://www.baidu.com")
public Data request(){

}
```

从这个例子里面可以看出，注解在一定程度上争强了扩展性和灵活性。

### Android Support Animation 的小例子

Android 里面的Resource Id都是int, 如果我们想要限制 Res 为Drawable

```java
public displayImage(int drawableRes) {
    // 显示图片
    // 如何确认 drawableRes 为drawable 的id?
}
```

```java

@Documented
@Retention(SOURCE)
@Target({METHOD, PARAMETER, FIELD})
public @interface DrawableRes {
}

public displayImage(@DrawableRes int drawableRes) {
    // @DrawableRes 可以用来表示是资源图片, 这个注解会由IDE进行检查
    // 当 drawableRes 不为drawable的时候，会提出相应的 warning
}
```