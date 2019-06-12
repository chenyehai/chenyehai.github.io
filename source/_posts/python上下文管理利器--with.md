---
title: python上下文管理利器--with
tags:
 - python
 - with
 - 上下文管理
abstract: 本文介绍python的上下文管理关键字with的基本用法，上下文管理协议，以及with语句的执行过程。然后介绍如何让对象支持上下文管理，最后介绍使用contextlib实现上下文管理的方法。
posturl: python-with
date: 2018-10-01 23:58:10
---

### with的基本语法与作用
with的语法(摘自python文档)如下：
```
with_stmt ::=  "with" with_item ("," with_item)* ":" suite
with_item ::=  expression ["as" target]
```
因为`with`语句可以包含多个`with_item`，所以可以同时使用多个上下文管理对象。
`with`用于包装使用了上下文管理对象的语句块的执行。使用了`with`关键字，在其包装的语句块执行完毕或者遇到异常时会自动做好善后清理工作（也就是执行将要在下文提到的`__exit__`方法）。把常见的异常处理用上下文管理对象来封装，将本来要[异常处理](/2018/10/01/python-exception/)中写在`finally`中的内容封装在`__exit__`方法中。这样使得代码更加`pythonic`。

### with带来的便利
如果不使用`with`,往一个文件里面写内容的代码可能长下面这个样子:
```python
try:
    file = open('test.txt', 'w+')
    file.write('hello world')
finally:
    file.close()
```
使用`with`的长这个样子:
```python
with open('test.txt', 'w+') as file:
    file.write('hello world')
```
显然，使用`with`代码更加优雅。在复杂的情况下，往往`with`能保证更高的正确率。

### 上下文管理协议

实现了上下文管理器协议的对象就被称为上下文管理器。上下文管理器协议比较简单：要求包含 __enter__ ，__exit__ 两个方法。所以简单解释如果一个对象实现了`__enter__`和`__exit__`两个方法，那么这个类的对象实例可以称为上下文管理对象。其中`__exit__`需要接受三个参数`exc_type`, `exc_value`和 `traceback`，它们分别代表异常类型、异常内容和异常的位置追踪信息。

### with语句的执行过程
`with`语句的执行过程大概如下：
1. 获取上下文管理对象。也就是执行上面语法中`with_item`中的`expression`。
2. 加载管理对象的`__exit__`方法，以便后面调用
3. 调用管理对象的`__enter__`方法，如果上面的`with_item`有`as target`部分，则将`__enter__`方法的返回值赋给target变量。
4. 执行主要语句块，也就是`suite`部分。
5. 调用上下文管理对象的`__exit__`方法。如果上一步没有异常，则三个参数都传`None`，否则就是异常类型、异常内容和异常的位置追踪信息
如果`suite`中没有异常，则一切正常。如果有异常，则取决于`__exit__`的返回值，如果`__exit__`的返回值为`False`，则异常会再次触发抛出，如果返回值为`True`，则异常不会抛出，后续代码继续执行。

### 让对象支持上下文管理
为了让对象支持上下文管理，只要让对象遵循上面说的上下文管理协议即可。下面我写一个支持上下文管理的类。
```python
class ContextDemo(object):
    def __init__(self):
        self._dict = {}

    def set_attr(self, k, v):
        self._dict[k] = v

    def get_attr(self, k):
        return self._dict[k]

    def __enter__(self):
        return self

    def __exit__(self, exc_type, exc_value, traceback):
        if exc_type:
            print(exc_type)
            print(exc_value)
            # return False
        return True

def test_my_context_demo():
    with ContextDemo() as demo:
        demo.set_attr('name', 'my_name')
        demo.get_attr('no')
test_my_context_demo（）
```
运行上面代码会输出以下两行内容：
`<type 'exceptions.KeyError'>`
`'no'`
因为with里面的语句块在调用`get_attr`时， 出现了错误，`no`不在字典中，所以调用`__exit__`时，`exc_type`会传递异常类型。最后因为是`__exit__`的返回值为True,所以执行代码时不会看到程序报异常，如果把`return False`的注释去掉，则异常还会往外抛。

### 使用contextlib实现上下文管理
如果你觉得像上面那样自己定义一个类有点麻烦，还有另一种简单的方法。旧式通过`contextlib`模块里的`contextmanager`装饰器装饰将函数装饰成为上下文管理器。函数中需要包含`yield`, 函数被`yield`关键词分割成两部分。

在`yield`之前执行的代码，会在上面所说的`with`语句执行过程的第一步执行，然后`yield`返回的值相当于`__enter__`的返回值。`yield`之后的代码，则在`with`语句块执行完毕之后再执行了。
当然了，被`contextmanager`只能有一个`yield`，也就是只能用next迭代一次，否则会报错。如果感兴趣，可以看看`contextlib.py`的源码。
下面是一个例子:
```python
from contextlib import contextmanager


@contextmanager
def simple_context_demo():
    try:
        print('get context manager obj')
        d = {'a': 123}
        yield d
    except KeyError as e:
        print(e)
    finally:
        print('in finally')


def test_context_manager_usage():
    with simple_context_demo() as ctx:
        print(ctx['a'])
```
执行结果如下:
>get context manager obj
>123
>in finally

结合执行结果，可以更加清晰的看出执行过程。

### 主要使用场景
我个人认为，`with`主要使用在资源的获取与释放。由于`with`的上下文管理机制，资源在使用完毕或者出现异常时，都会释放并且执行必要的清理。比如锁、文件操作等会用的比较多。`python`线程摸块的锁就实现了上下文管理协议，支持`with`关键字直接使用。