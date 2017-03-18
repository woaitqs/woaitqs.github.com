---
layout: post
title: "算法面试精要准备 - 栈"
description: "说明一些常见面试题，关于栈的技巧，帮助你在面试时有所斩获"
category: "program"
keywords: "算法, Stack, 栈"
tags: [算法, 动态规划, 数据结构, 栈]
---
{% include JB/setup %}

栈是什么呢？很简单，就是实现后进先出(LIFO)的数据结构，在某些算法题中，就会利用栈的这种特性来达成目的。所有代码都已经同步至 [https://github.com/woaitqs/common_algorithm](https://github.com/woaitqs/common_algorithm)，欢迎关注。

<!--break-->

### 0X000 例子1

我们从最直接的例子上入手，看看栈是如何帮助我们的。

原题来源：[leetcode-20](https://leetcode.com/problems/valid-parentheses/)

> Given a string containing just the characters '(', ')', '{', '}', '[' and ']', determine if the input string is valid.The brackets must close in the correct order, "()" and "()[]{}" are all valid but "(]" and "([)]" are not.

题目大意是，给出一个包含 '(', ')', '{', '}', '[' 和 ']' 的字符串，判断这个字符串是否符合数学常识，也就是是否成双成对。

利用栈的特性，将属于带有左性质(左小括号，左大括号等等)的符号入栈，当遇到右性质的符号就出栈，同时比较两个符号是否成对。

```java
public boolean isValid(String s) {

    if (s == null || s.isEmpty()) {
        return true;
    }

    Stack<Character> characters = new Stack<>();

    for (char item : s.toCharArray()) {
        if (item == '(' || item == '{' || item == '[') {
            characters.push(item);
        } else {
            if (characters.isEmpty()) {
                return false;
            }

            char top = characters.pop();
            switch (item) {
                case ')':
                    if (top != '(') {
                        return false;
                    }
                    break;
                case '}':
                    if (top != '{') {
                        return false;
                    }
                    break;
                case ']':
                    if (top != '[') {
                        return false;
                    }
                    break;
                default:
                    return false;
            }
        }
    }
    return characters.isEmpty();
}
```

### 0X001 例子2

原题来源：[leetcode-94](https://leetcode.com/problems/binary-tree-inorder-traversal/)

二叉树的中序遍历题目，一般有递归方式与非递归方式两种。这里不介绍递归方式，而是用栈的特性，来帮助我们完成遍历。

1. 维护一个栈 stackNodes 和一个当前节点 traversalNode。初始时将 traversalNode 赋值为根节点。

2. 将当前节点 traversalNode 的左子链上的节点逐个推入栈中，直到没有左儿子。

3. 取出栈顶的节点，访问该节点，将 traversalNode 赋值为该节点的右儿子。

4. 不断执行 2，3，直到栈为空且当前节点也为空。


```java
private List<Integer> inorderTraversalV2(TreeNode root) {
    List<Integer> values = new ArrayList<>();
    Stack<TreeNode> stackNodes = new Stack<>();
    TreeNode traversalNode = root;
    while (traversalNode != null || !stackNodes.isEmpty()) {
        while (traversalNode != null) {
            stackNodes.add(traversalNode);
            traversalNode = traversalNode.left;
        }
        TreeNode currentNode = stackNodes.pop();
        values.add(currentNode.val);
        if (currentNode.right != null) {
            traversalNode = currentNode.right;
        }
    }
    return values;
}
```

### 0X002 例子3

最后再来看看一个复杂的例子。

原题来源：[leetcode-84](https://leetcode.com/problems/largest-rectangle-in-histogram/)

> Given n non-negative integers representing the histogram's bar height where the width of each bar is 1, find the area of largest rectangle in the histogram.

![histogram area](http://o8p68x17d.bkt.clouddn.com/histogram_area.png)

给定一个非负整数的数组，这个数组如上图一样，可以用直方图的形式来展示，求能够组成的最大矩形的面积。在上图中，红色矩形的那一块就是最大的面积。

这个题目也是对栈的利用。我们先想想一个递增的序列，在这种情况下，矩形最大的面积一定是可以连续计算的，但一定遇到下跌的数字，出现了断层，这一块区域就不能连续计算了。由此我们通过栈的方式，来维持一个递增序列，当遇到下降时就进行计算一次。这里的逻辑相对复杂，大家需要多花一些时间来理解。

```java
public int largestRectangleArea(int[] heights) {
    int maxArea = 0;
    List<Integer> newHeights = new ArrayList<>();
    for (int item : heights) {
      newHeights.add(item);
    }
    newHeights.add(0);

    Stack<Integer> areaIndexs = new Stack<>();

    int index = 0;
    while (index < newHeights.size()) {
      if (areaIndexs.isEmpty() || newHeights.get(index) > newHeights.get(areaIndexs.peek())) {
        // push index to stacks.
        areaIndexs.push(index);
        index ++;
      } else {
        int top = areaIndexs.peek();
        areaIndexs.pop();
        maxArea = Math.max(
            maxArea,
            newHeights.get(top) * (areaIndexs.isEmpty() ? index : index - areaIndexs.peek() - 1));
      }
    }
    return maxArea;
  }
```
