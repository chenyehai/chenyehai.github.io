---
title: python类的继承与方法解析顺序（MRO）
tags:
 - python
 - oop
 - MRO
abstract: 本文讲述python的新式类与经典类的方法解析顺序（MRO）的区别，新式类MRO采用C3算法及算法过程，以及新式类不允许的继承情况等。
posturl: python-oop-mro
date: 2018-10-02 15:31:19
---



### 经典类与新式类
python中的类有两种，一种是`new-style classes`，业界翻译为**新式**类；另一种是`classic classes`，业界翻译为（经典类、旧式类、古典类），本文统一称之为经典类。
新式类是从python2.2版本出现，新式类在2.2版本中使用的方法解析顺序（以下简称MRO）与2.3及以后的版本不同。本文说的新式类MRO是指2.3版本及以后的。2.2版本的与经典类的相同，本文不细说（因为如今实际开发中都使用2.7或者3系列的版本了）。
新式类如何定义呢？
在python2系列（2.3版本后）里面，只要基类包含`object`的类就是新式类。所以定义新式类就是集成于一个新式类或者`object`就可以了。内建类型（dict,list）也是新式类。
在python3中已经废除了经典类，默认定义就是新式类。也就是怎么定义都是新式类。

### 新式类与经典类的区别
新式类与经典类最大的区别在于MRO不同。除了MRO不同外，其他方面的细小区别这里列举一些常见的有用的，但不深入探讨:
1. 新式类有`__slots__ `属性，用于指定类的对象拥有的属性，对象不维护一个属性字典，节约内存空间。
2. 新式类及其对象可以作为`super`内建函数的参数，经典类是不可以的。
3. 新式类有`__mro__`属性，访问它返回解析顺序列表。

### 采用C3以前的MRO
python2.3及以后的版本采用C3算法实现MRO。由于我对python2.2的新式类及python的经典类采用的MRO算法没有考究得很清楚（没看到文章说python2.2的新式类与经典类MRO的区别），因此这里不说经典类的MRO,也不说是python2.2的MRO，而是说C3以前的MRO。
它们的解析顺序大概就是深度优先，从左到右进行搜索，第一个搜索到属性或方法生效。

### 采用C3算法的MRO
采用C3算法以前的MRO存在一些问题，见[官网](https://www.python.org/download/releases/2.3/mro/)。简单说，就是存在两个问题，一是不能保持局部优先级，也就是不能遵循继承顺序；二是不能保持单调性（monotonic），官网定义为` A MRO is monotonic when the following is true: if C1 precedes C2 in the linearization of C, then C1 precedes C2 in the linearization of any subclass of C.`，也就是说子类的MRO不能与基类的MRO相悖（如c1与c2的相对顺序，基类：C1 -> C2，子类：C2 -> C1，这是相悖的）。这些问题在多继承的情况下可能出现。
采用了C3算法后的MRO是完美的（至少目前业界认可）。那么C3算法的是怎么样的呢？官网上有代码，但是不好懂。我结合实例，用我的理解说说。
```python
class A(object):
	pass


class B(A):
	pass


class C(A):
	pass


class D(B):
	pass


class E(B, C):
	pass


class F(D, C):
	pass


class G(F, E):
	pass
```
如上面代码，G的MRO，用mro(G)表示, mro最终是一个序列[]。公式表示如下:
```
mro(G) = [G] + merge(mro(F), mro(E), [F, E])
```
就是它本身加上各个直接基类的mro与基类顺序序列的merge结果。因为G直接继承了F和E，所以上式基类顺序序列就是[F,E]（按照它们继承的顺序）。

由于merge里面包含了基类的mro，因此需要递归求出所有直接基类的mro，然后merge。

merge是C3算法的核心。merge的过程是：**按顺序**遍历需要merge的序列，如果一个序列的第一个元素，在**它之前的序列中不存在**，在它**之后**的序列中或者是第一个元素，或者不存在，则将该元素合并到当前已求出的序列中，并且从所有merge操作的序列中删除这个元素。按顺序的意思就是从左到右遍历，不是随机取，第一个序列中如果没有符合条件的元素，则从第二个序列找，以此类推。
mro求解过程是递归的，下面列出的mro(F)的求解过程如下:
```
下面object简写为O，故有mro(O) = [O]
mro(F) = [F] + merge(mro(D), mro(C), [D,C])
递归求mro(D),
mro(D) = [D] + merge(mro(B), [B])
递归求mro(B),
mro(B) = [B] + merge(mro(A), [A])
递归求mro(A),
mro(A) = [A] + merge(mro(O), [O])
       = [A] + merge([O], [O])
       = [A, O]
求得mro(B)
mro(B) = [B] + merge([A, O], [A])
       = [B, A] + merge([O])
       = [B, A, O]
求得mro(D)
mro(D) = [D] + merge([B, A, O], [B])
       = [D, B] + merge([A, O])
       = [D, B, A, O]
求得mro(C)
mro(C) = [C] + merge(mro(A), [A])
       = [C] + merge([A, O], [A])
       = [C, A] + merge([O])
       = [C, A, O]
求得mro(F)
mro(F) = [F] + merge([D, B, A, O], [C, A, O], [D, C])
       = [F, D] + merge([B, A, O], [C, A, O], [C])
       = [F, D, B] + merge([A, O], [C, A, O], [C])
       = [F, D, B, C] + merge([A, O], [A, O])
       = [F, D, B, C, A] + merge([O], [O])
       = [F, D, B, C, A, O]
```
上面过程中，选取最后求mro(F)合并时的过程讲解：
1. 因为D在第一个序列[D, B, A, O]的第一位，而第二个序列[C, A, O]中不包含D，第三个序列[D, C]中D也在第一位。因此，将D添加到已知序列[F]中，并从merge的三个序列中删除。得到`[F, D] + merge([B, A, O], [C, A, O], [C])`
2. 将B添加到已知序列，并从merge的三个序列删除，得到`[F, D, B] + merge([A, O], [C, A, O], [C])`。此处不能先移除C，因为第一个序列[B, A, O]优先，并且B也符合条件。
3. 这一步第一个序列中的A不符合移除条件，因为它出现在第二个序列中并且不是第一位。检查第二个序列，第二个序列C符合条件。将C添加到已知序列，并从merge的三个序列中删除C， 得到`[F, D, B, C] + merge([A, O], [A, O])`。
4. 依次移除添加A,O得到结果。

同理可以求得`mro(E) = [E, B, C, A, O]`，进而求得`mro(G) = [G, F, D, E, B, C, A, O]`。
大家可以设计一些其他继承情形，写代码练习验证，看看用代码输出类的`__mro__`是否与自己想的一致。
比如上面代码调用的`print(G.__mro__)`输出如下：
```
(<class '__main__.G'>, <class '__main__.F'>, <class '__main__.D'>, <class '__main__.E'>, <class '__main__.B'>, <class '__main__.C'>, <class '__main__.A'>,<type 'object'>)
```

### 新式类不允许的继承情形
**那么问题来了**，按照这个算法，肯定存在一些情况是merge过程最终无法完成的，那么它们的mro是什么呢？事实上，这种情况所对应的继承情形在python2.3及以后版本中的新式类是不允许的。比如如下代码，在编译时将会报错，报错信息提示`Cannot create a consistent method resolution`
```
class A(object):
	pass


class B(A):
	pass


class C(A, B):
	pass
```
这个报错是编译时的，如果提示这个错误就需要改类的继承关系了。那么有没有简单的判断规则呢？我觉得就是把所有类看作顶点，把继承关系看作有向边，把所继承基类的mro（上例的 B --> A --> O）以及它的继承顺序(先继承A,再继承B, 继承顺序为A --> B)的边集合能不能构成一个有向无环图。如果可以，那么继承合法，如果图有环，则不允许出现该继承类型。上例中顶点A和B显然存在环，故不合法。

### 新式类MRO带来的优势
由于新式类的MRO采用了C3算法，比较完善，在多继承时就有了优势，同一个类的方法不会被重复调用。比如下面代码菱形继承的情形：
```python
class A(object):
    def say_something(self):
        print('I am in A')

class B(A):
    def say_something(self):
        print('I am in B')
        super(B, self).say_something()

class C(A):
    def say_something(self):
        print('I am in C')
        super(C, self).say_something()

class D(B, C):
    def say_something(self):
        print('I am in D')
        super(D, self).say_something()

d = D()
d.say_something()
```
运行代码输出如下：
```
I am in D
I am in B
I am in C
I am in A
```
A虽然被B、C继承了，但是它的方法只执行一次，不会重复调用。
假如使用经典类，首先不能用super，然后就要手动指明调用哪个类的方法，可能会出错。

### 其他
MRO虽然是方法解析顺序，但并不是仅仅是类方法按照这个顺序查找，查找类的属性也是遵循这个顺序的。

### 总结
1. 不是特殊情况都使用新式类，由于新式类的MRO使用了C3算法，会限制一些不合理的继承情形，所以使用新式类有助于促进我们设计更科学的继承。比如直接继承一个基类两次是不允许的。
2. 需要向上调用基类方法时，最好使用super，这样同一个类的方法不会重复调用。如果真的不想用super，那就统一全部不用super，混用是不对的。 