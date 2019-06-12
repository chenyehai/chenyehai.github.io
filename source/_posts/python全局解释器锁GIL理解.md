---
title: python全局解释器锁GIL理解
tags:
 - python
 - GIL
abstract: 本文结合我自己的理解简单说一下由C语言实现的CPython解释器中的GIL相关的问题。由于本人实力不足，研究不深，文中所述有误还请各位指正，大神轻喷。
posturl: python-gil
date: 2018-10-01 15:46:47
---

### 基本概念
python是解释型脚本语言，所以运行python代码时需要一个解释器来将`py`源代码编译成字节码也就是`pyc`文件，然后再由其虚拟机来执行`pyc`文件中的字节码。
目前python的解释器有很多种实现，最为常见的是C语言实现的CPython解释器，此外还有Java语言实现的Jython、python语言实现的PyPy等解释器。
而我们说的GIL(global interpreter lock)是存在于CPython解释器中，并不是python语言本身的特性。所以，本文所有的说法都是基于C语言实现的CPython解释器。

### GIL的作用及其影响
下面引用官方文档的一段话，表达了GIL机制的作用。
>The mechanism used by the CPython interpreter to assure that only one thread executes Python bytecode at a time. This simplifies the CPython implementation by making the object model (including critical built-in types such as dict) implicitly safe against concurrent access. Locking the entire interpreter makes it easier for the interpreter to be multi-threaded, at the expense of much of the parallelism afforded by multi-processor machines.

从引文中可以看到，GIL真的就是一把锁，用于保证CPython解释器中同一时间只有一个线程在执行python的字节码。使用了GIL，使得解释器很容易实现支持多线程的特性，但是在多核处理器中会牺牲并行特性，因为整个解释器进程中只允许一个线程在执行python的字节码。
所以GIL是一把粗粒度的锁,是全局排他锁，把整个解释器锁起来。既然是锁，那么用了它之后，对性能必然是会有一定的损耗。这是它带来的影响。

### GIL何时释放
GIL是一把锁，那么它就有获得锁与释放锁的过程。python线程在需要用到解释器时就需要获得锁，线程在以下三种情况会释放GIL：
1. 线程的时间片完了会释放。在python2中默认是执行100个python字节码指令会把GIL释放一次，可以调用`sys.setcheckinterval`来设置检查释放周期。在`python3.2`以后，建议使用`sys.setswitchinterval`方法设置时间片的秒数。
2. 在发生I/O操作时，GIL会释放。
3. 调用C代码时，不需要用到解释器，也会释放GIL。比如使用ctypes模块中的api时。如果是自己写的C扩展，使用`Py_BEGIN_ALLOW_THREADS`释放GIL，还要用`Py_END_ALLOW_THREADS`来重新获取GIL，也就是把不需要使用到python解释器的代码夹在这两个宏之间。

### 如何绕过GIL
GIL是个历史遗留的问题，要想把CPython解释器的实现中的GIL去除工作量很大。历史上也曾试过移除，但是移除之后性能更加差，所以也就放弃了移除。
由于在CPython解释器中移除GIL非常困难，也许需要我们在开发中注意它的负面影响。
绕过它的方法有几种：
1. CPU密集型的任务使用多进程。由于每个进程有自己的解释器，所以他们的GIL是独立于进程的。这样也可以充分利用多核CPU的性能。
2. GIL在I/O过程中会释放，因此在I/O密集型的程序中使用多线程的性能会比单线程的好。也就无需绕过。
3. 由于GIL只是锁定CPython的解释器，在调用C代码时会释放，因此在可以使用C语言扩展来实现对性能要求比较高或者使用频繁的模块。
4. 通过设置合理的线程调度周期来减少线程的切换时间，也有利于性能的提高。因为GIL在释放的过程中需要将线程的状态进行保存，频繁切换需要付出代价。
5. 还可以尝试使用其他语言实现的解释器，如Jython。

### 小结
由于GIL的客观存在，并且CPython中还没有找到理想的去除方式。我们没有能力去改变，不如以更好的方式去接受。比如：
1. I/O 密集型任务（如网络爬虫、文件处理等）使用多线程实现。
2. CPU密集型任务(如科学计算等)使用多进程实现。
3. 对性能要求较高的核心代码使用C语言实现。