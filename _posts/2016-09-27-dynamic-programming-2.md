---
layout: post
title: "也来谈谈动态规划-2"
description: "从简单的例子入手分析动态规划，明白动态规划的原理"
keywords: "动态规划, 详细讲解"
category: "program"
tags: [算法, 动态规划]
---
{% include JB/setup %}

今天刚好遇到两个动态规划的例子，分别来自于 去哪儿 和 涂鸦的面试题，这两个题目都涉及到简单的动态规划，这里整理一下，作为对 [也来谈谈动态规划](http://www.woaitqs.cc/program/2016/04/22/dynamic-programming) 的补充。

<!--break-->

### 个数最小完全平方和

> 给一个正整数 n, 找到若干个完全平方数(比如1, 4, 9, ... ) 使得他们的和等于 n。你需要让平方数的个数最少。
给出 n = 12, 返回 3 因为 12 = 4 + 4 + 4。
给出 n = 13, 返回 2 因为 13 = 4 + 9。

同样，我们来构建状态迁移方程, 后文中的 sqrt 表示平方根，floor 表示向下取整。

对于 1 而言，只有 floor(sqrt(1)) = 1，刚好 1 * 1 = 1，所以这里就是 1, f(1) = 1.

对于 2 而言，floor(sqrt(2)) = 1, 这里 1 * 1 < 2, 因此 1 是不够的，只是其中的一部分，所以个数应该是 f(2) = 1 + f(2 - 1*1))

对应 3 而言，floor(sqrt(3)) = 1, 这里 1 * 1 < 2, 所以 1 还是不够，计算方法是 f(3) = 1 + f(3 - sqrt(1*1))

对于 4 而言，floor(sqrt(4)) = 2, 这里 2 * 2 = 4，所以 2 的平方和是 4，刚好足够，f(4) = 1

对于 5 而言，floor(sqrt(5)) = 2, 而 2 * 2 < 4，这里 2 还是不够，这里重点来了。f(5) = 1 + f(5 - 2*2)), 这个式子对吗？
不对，因为 floor(sqrt(5)) = 2，2 下面还有 1，也就是说 f(5) 这里有了两个选择，5 = 2 * 2 + ?, 也可以是 5 = 1 * 1 + ？。
除了 f(5) = 1 + f(5 - 2*2)), 也可以等于 f(5) = 1 + f(5 - 1*1))
我们需要选择两个中的最小值。

好了，我们来总结一下状态转移方程。f(num) = min{1 + f(num - seed * seed)}, 其中 seed 取值为「floor(sqrt(num)) , ... , 3, 2, 1」。知道原理过后，代码就比较简单了。

```java
public static int leastSquareSum(int num) {
    int maxOne = (int) Math.sqrt(num);
    if (maxOne * maxOne == num) {
        return 1;
    }

    int result = Integer.MAX_VALUE;
    while (maxOne > 0) {
        int currentSize = 1 + leastSquareSum(num - maxOne * maxOne);
        result = result < currentSize ? result : currentSize;
        maxOne --;
    }

    return result;
}
```

### 酒店最少天数

> 你要去旅游，有N元的预算住酒店，有M家酒店供你挑选，这些酒店都有价格X。需要你 **正好** 花完这N元住酒店（不能多，也不能少），请问最少能住几晚？返回最少住的天数，没有匹配的返回 -1。

> 比如你有1000，所有酒店都是大于 1000 的，返回 -1
比如你有1000元，有一家1000元的，一家300元的，一个700元的，则最少能住1晚
比如你有1000元，有一家1元的，一家2元的，一家1001元的，则最少能住500晚。

这个题目，大体思路上和上一题差不多，需要做一些变通。思路可以这么想想，如果我们要找最少的酒店天数，那么一定尽量选价格贵的，如果全选价格贵的后，不足以刚好匹配钱，那么剩下的再从次高的价格中去补。基于前面的思路后，状态方程就比较好写了。

f(num) = min{item + f(money - item * prices(i))}; 其中 item 取值为 1, 2, 3, ..., floor(money/prices(i))), prices(i) 为第 i 个酒店的价格, i 取值为 1，2, ..., 酒店价格数目

```java
// 这里的prices是经过降序处理后的
public static int findLeastNum(int money, List<Integer> prices) {
    if (prices.size() == 0) {
        return -1;
    }
    if (prices.size() == 1) {
        if (money % prices.get(0) == 0) {
            return money / prices.get(0);
        } else {
            return -1;
        }
    }
    for (Integer price : prices) {
        if (money % price == 0) {
            return money / price;
        }
        int preNum = money / price;

        int tryNum = preNum;
        while (tryNum > 0) {
            List<Integer> subPrices = prices.subList(1, prices.size());
            int lastPartNum = findLeastNum(money - tryNum * price, subPrices);
            if (lastPartNum != -1) {
                return tryNum + lastPartNum;
            }
            tryNum --;
        }
    }
    return -1;
}
```

------------------------

### 文档信息
* 版权声明：自由转载-非商用-非衍生-保持署名（[创意共享3.0许可证](http://creativecommons.org/licenses/by-nc-nd/3.0/deed.zh)）
* 发表日期：2016年9月27日
* 社交媒体：[weibo.com/woaitqs](http://weibo.com/woaitqs)
* Feed订阅：[www.woaitqs.cc/feed.xml](http://www.woaitqs.cc/feed.xml)

------------------------
