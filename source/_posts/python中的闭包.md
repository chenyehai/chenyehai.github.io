---
title: python中的闭包
tags:
 - python
 - 闭包
abstract: 本文介绍闭包的概念、闭包的使用场景以及使用闭包需要注意的问题。大部分高级编程语言（如JavaScript、python）都支持闭包的特性，本文以python语言作为例子，举例说明。
posturl: python-closure
date: 2018-09-30 21:12:25
---

### 闭包的概念

闭包是由函数以及创建该函数的词法环境组合而成。这个环境包含了这个闭包创建时所能访问的所有局部变量。
下面是引用自[维基百科](https://zh.wikipedia.org/wiki/%E9%97%AD%E5%8C%85_%28%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%A7%91%E5%AD%A6%29)的一段话，
`在计算机科学中，闭包（英语：Closure），又称词法闭包（Lexical Closure）或函数闭包（function closures），是引用了自由变量的函数。这个被引用的自由变量将和这个函数一同存在，即使已经离开了创造它的环境也不例外。所以，有另一种说法认为闭包是由函数和与其相关的引用环境组合而成的实体。闭包在运行时可以有多个实例，不同的引用环境和相同的函数组合可以产生不同的实例。
`
我的理解：闭包就是在一个函数（姑且称作外层函数）中定义了另一个函数（内层函数），并且内层函数中引用了外层函数的一个或多个局部变量。通常，外层函数返回值为内层函数。

### 闭包实例与理解

下面是一个python语言的闭包实例。
```python
def my_pow(x):
	def inner_pow(y):
		return x**y
	return inner_pow

pow3 = my_pow(3)
pow4 = my_pow(4)

print pow3(2) # 输出3的2次方，9
print pow4(2) # 输出4的2次方，16
```
在上面例子中，pow3实际就是inner_pow与my_pow中的引用环境复合而成，也就是维护了my_pow中的x的值为3。pow3和pow4中的x值是互不干扰，单独维护的，所以调用pow3和pow4能得到正确的结果，它们的引用环境是不同的。

### 使用闭包的场景

进行惰性计算时可以使用闭包。因为惰性计算可能需要延迟调用闭包内的函数，并且要在局部维持一些变量。
由于闭包实现了封装，某些需要用到类的场景可以使用闭包实现，但是一般建议是属性与方法比较少时，才用闭包代替类。
总的来说，就是在需要维护一个非全局变量时使用闭包，将需要维护的非全局变量定义在创造闭包的环境中（闭包外层函数的局部变量）即可。比如下面的例子，实现计时器，需要维护总次数`total`，但是总次数不是全局变量,借助闭包实现容易。
```python
def counter():
	total = 0
	def incr():
		nonlocal total #  nonlocal关键字在python3中才有
		total += 1
		return total

	return incr
```
上面例子用到`nonlocal`关键字，该关键字python2没有，故在python2中需要换一种方式，比如将次数记录在列表a的第一个元素中,初始化为`a=[0]`, 每次增加的时候就是`a[0] = a[0] + 1`。

### 使用闭包需要注意的问题

1. python闭包中如果需要修改外部函数的局部变量，需要格外注意。如上面计时器的例子，需要加入`nonlocal`关键字表明`total`不是内层函数的局部变量，需要往上查找。如果没有这一行，则total为内层函数的局部变量，会导致结果不符合预期。
2. 由于闭包的内层函数需要引用外层函数的变量，因此在外层变量会发生变化时需要留意。比如下面的例子：
```python
def func_list():
	result_list = []

	def inner_func():
		return i * 2

	for i in range(4):
		result_list.append(inner_func)
    
	return result_list

l = func_list()
for fc in l:
	print(fc())
```
这个例子中，每次调用fc返回的都是6。因为`inner_func`绑定的外层函数变量i的值是在调用`inner_func`的时候才查找的，在调用它的时候，i的值是3，故每次返回值都是6。上面例子还可以写成`lambda`的形式，`result_list = [lambda :i*2 for i in range(4)]`。形式的改变，不会影响它的运行结果，因为`lambda`只是函数的另一种写法。

3. 如果在闭包中维护过多的变量，对性能或多或少有影响，所以闭包应该尽量精巧。