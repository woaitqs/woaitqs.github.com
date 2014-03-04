---
layout: post
title: "try和finally来干一仗"
description: "try finally block"
category: "code"
tags: [code]
---
{% include JB/setup %}

- try, catch 和 finally是java常见的语句块，但里面的语法规则确实非常复杂的。在一个try的语句块中含有finally时执行的语句规则如下。

- 如果在try的语句块顺利地执行完，那么finally的部分会被执行分为如下两种情况。
> - 如果finaly部分执行完成，那么`try-catch`语句块执行完成。
> - 如果finaly部分因为S原因而抛出异常，那么`try-catch`以S异常而突然结束。

- 如果try部分的代码因为V异常而突然停止，那么情况如下
> - 如果try抛出的V异常是catch模块所申明的情形，那么catch部分代码会被执行
> - 如果catch部分代码顺利执行完，那么finally部分代码会被执行。
> - 如果finaly部分执行完成，那么`try-catch`语句块执行完成。
> - 如果finaly部分因为S原因而抛出异常，那么`try-catch`以S异常而突然结束。
> - 如果catch部分以R异常突然结束，那么finally部分代码会被执行。
> - 如果finaly部分执行完成，那么`try-catch`以R异常结束
> - 如果finaly部分因为S原因而抛出异常，那么`try-catch`以S异常而突然结束。
> - 如果try抛出的V异常未被catch模块所申明，那么情况如下
> - 如果finaly部分执行完成，那么`try-catch`以V异常结束。
> - 如果finaly部分因为S原因而抛出异常，那么`try-catch`以S异常而突然结束。

- 例子如下

{% highlight java %}
class BlewIt extends Exception {
        BlewIt() { }
        BlewIt(String s) { super(s); }
}
class Test {
        static void blowUp() throws BlewIt {
                throw new NullPointerException();
        }
        public static void main(String[] args) {
                try {
                        blowUp();
                } catch (BlewIt b) {
                        System.out.println("BlewIt");
                } finally {
                        System.out.println("Uncaught Exception");
                }
        }
}
{% endhighlight java %}

输出的结果

```
Uncaught Exception
java.lang.NullPointerException
        at Test.blowUp(Test.java:7)
        at Test.main(Test.java:11)
```
