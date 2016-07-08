---
layout: post
title: "Android Annotation 实战"
description: "详细说明 Android 注解"
keywords: "annotation, android, 注解，详细说明"
category: "android"
tags: [annotation, android]
---
{% include JB/setup %}

## 什么是注解

Java 注解又成为 Java 标注， 是 Java 语言在5.0版本以上开始支持加入源代码的特殊语法元数据。

我们常见的 @Override， @Despited 等这些都是注解，注解是一种可以在 Runtime， Source， Class 三个级别上来提供给我们进行语义表达的特殊数据格式。

JDK 定义了四中基础的注解元数据，分别是 @Documented, @Retention, @Target, @Inherited。

<!--break-->

1. @Documented 用于描述其它类型的annotation应该被作为被标注的程序成员的公共API
2. @Retention 定义了该Annotation被保留的时间长短：某些Annotation仅出现在源代码中，而被编译器丢弃；而另一些却被编译在class文件中；编译在class文件中的 Annotation 可能会被虚拟机忽略，而另一些在class被装载时将被读取（请注意并不影响class的执行，因为Annotation与class在使用上是被分离的）。
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

### Android Support Annotation 的小例子

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

### 动手开始实现一个Annotation

#### 需求描述

```java
public interface GitHubService {
  @GET("/users/{user}/repos")
  List<Repo> listRepos(@Path("user") String user);
}
```

这是从RetroFit里面截取的一段代码，里面涉及到两个 Annotation， 一个是 @PATH， 一个是 @GET, 分别用来标记
使用 GET 方法来获取内容， 另一方面是 通过 PATH 来指定 URL 里面的参数， 实现动态可替换。

GET 注解需要在 运行时 也可以获取，这样我们才能在运行时，拿去 URL 进行相应的操作，因而需要指定 Retention 为 RUNTIME, 另一方面，GET 只能用于修饰某个方法，因此 Target 需要为 METHOD。 那么我们如何对这个注解赋值呢？ 需要告诉 GET 注解需要的URL， 注解是可以定义方法的， 因而我们这里实现了 value() 方法， 默认值为空.

```java
/** Make a GET request to a REST path relative to base URL. */
@Documented // 文档可描述
@Target(METHOD)
@Retention(RUNTIME)
public @interface GET {
  String value() default "";
}
```

同理，PATH 注解的定义如下

```java
@Documented
@Retention(RUNTIME)
@Target(PARAMETER)
public @interface Path {
  String value();

  /**
   * Specifies whether the argument value to the annotated method parameter is already URL encoded.
   */
  boolean encoded() default false;
}
```

完成这个两个注解后，我们来看看将这两个注解用起来。java 反射包里面实现了大部分的功能，通过这个包里面提供方法，可以达到我们的目的。

```java
for (Annotation annotation : method.getAnnotations()) {
    // 通过 method.getAnnotations() 可以获得修饰方法的注解
    if (annotaiton instance GET) {
        // 处理注解是GET方法的情况
        String url = ((GET) annotation).value();

    }
}

if (methodParameterAnnotation instanceof Path) {
    addPathParam(((PATH) annotation).value(), argu);
}

void addPathParam(String name, String value, boolean encoded) {
    if (relativeUrl == null) {
      // The relative URL is cleared when the first query parameter is set.
      throw new AssertionError();
    }
    try {
      if (!encoded) {
        String encodedValue = URLEncoder.encode(String.valueOf(value), "UTF-8");
        // URLEncoder encodes for use as a query parameter. Path encoding uses %20 to
        // encode spaces rather than +. Query encoding difference specified in HTML spec.
        // Any remaining plus signs represent spaces as already URLEncoded.
        encodedValue = encodedValue.replace("+", "%20");
        relativeUrl = relativeUrl.replace("{" + name + "}", encodedValue);
      } else {
        relativeUrl = relativeUrl.replace("{" + name + "}", String.valueOf(value));
      }
    } catch (UnsupportedEncodingException e) {
      throw new RuntimeException(
          "Unable to convert path parameter \"" + name + "\" value to UTF-8:" + value, e);
    }
}
```
