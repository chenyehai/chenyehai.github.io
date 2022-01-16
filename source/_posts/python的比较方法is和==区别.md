---
title: python的比较方法is和==区别
tags:
 - python
abstract: 本文简单讲一下python中比较两个对象的方法`is`和`==`的区别。
posturl: python-is-equal
date: 2018-10-01 00:10:18
---

python中经常使用到`is`和`==`比较两个对象，其实他们有一点细微的区别。这里用例子简单说一下。下图是在python 自带的idle中直接输入代码得到的结果：
![](/img/is_none_code.png)

1. a和b的值相等，并且指向同一个对象（***因为python对小整数做了缓存***），即`id`方法返回值相同，故`a is b`和`a == b`的结果都为`True`。
2. d和e指向的对象不同，故`d is e`的值为`False`.


下面引用文档中的一句话
>The operators **is** and **is not** test for object identity: x **is** y is true if and only if x and y are the same object. x **is not** y yields the inverse truth value. 

可见`is`是比较变量是否指向同一个对象。而`==`是比较对象的值，`==`调用的是`__eq__`方法。
