---
title: python的异常处理
tags:
 - python
 - 异常处理
abstract: 本文介绍python的try语句进行异常捕获与处理的几种常见形式，以及一些使用上要注意的问题。特别是在try以及finally子句中都有return时的要注意的坑。
posturl: python-exception
date: 2018-10-01 22:39:58
---

### 异常分类
我认为python中的异常分为两类，一类为编译时异常，另一类为运行时异常。
编译时异常是语法错误导致的，会提示语法错误，比如`SyntaxError: invalid syntax`。编译时异常只能是根据错误提示，将语法修改好了。
运行时是程序在运行中出现，运行时异常往往不可避免，因为除了程序本身的bug，它还可能是用户输入不符合预期而导致的。因此，我们需要对运行时异常进行捕获处理。

### python的异常处理语法
python的异常处理语法比较简单，不过与其他语言相比，有个特别的地方是（像python中的`for`和`while`一样）支持`else`子句，即`try-except-else`的形式。
下图是语法规则，摘自python文档。
![](/img/try_statement.png)

也就是python的异常处理有两种形式，一是`try-finally`的形式，后面不能跟其它子句；另一种是`try-except`的形式，后面可以跟一个`else`或`finally`子句（`else`和`finally`可以同时存在）。
`except`后面可以跟上具体的异常类型或者含有几种异常类型的元组作为捕获条件，只要引发的异常属于捕获条件中指定的类型或者其子类，异常即被匹配的`except`捕获。

### 实际例子
下面是实际例子的代码
```python
# -*- coding:utf-8 -*-

def without_inner_except_handle():
    """内层try没有捕获异常，在外层处理"""
    try:
        try:
            a = 3 /0
        finally:
            print('inner finally')
    except ZeroDivisionError as e:
        print('outer except match')
        print(e)


def without_except_handle():
    """没有处理的异常，遇到return、break时，异常信息则会丢弃不会往函数栈外抛"""
    try:
        3 / 0
    finally:
        return 1


def without_any_except():
    """没有任何异常，则执行else语句块"""
    try:
        a = 3 / 4
    except:
        print('except found')
    else:
        print('no except found')
    finally:
        print('excute finally')


def return_in_try_and_finally():
    """即使return在try中，finally也会执行。"""
    try:
        return 'try'
    except:
        return 'except'
    else:
        return 'elsel'
    finally:
        return 'finally'


if __name__ == '__main__':
    split_line = '=' * 50
    without_inner_except_handle()
    print(split_line)

    result = without_except_handle() 
    print(result)
    print(split_line)

    without_any_except()
    print(split_line)

    result = return_in_try_and_finally()
    print(result)

```
以上代码执行结果如图:![](/img/try_except_result.png)

### else与finally的区别
 `else`子句是在没有任何异常时执行。如果有异常，即使没有被匹配的`except`捕获到，`else`子句的代码也不会执行。
`finally`则无论如何都会执行，可以认为它专门用来做清理工作。


### 需要注意的问题
结合上面的例子理解，需要注意的问题有两点：
1. 无论如何，`finally`一定会执行。因此，如果`try`或者其他子句与`finally`中均有`return`，那么只有`finally`中的`return`会执行。也就是返回值由`finally`决定。见上例的`return_in_try_and_finally`函数的返回结果。（*<u>这一点与实现方式有关，未来可能会改变</u>*）
2. 如果捕获异常的代码中有`return`或者`break`，未处理的异常会被丢弃，否则未处理的异常会往上抛。
3. 结合上一点，应该在异常捕获代码中留一个可以捕获所有异常的`except`语句，避免异常往外抛.