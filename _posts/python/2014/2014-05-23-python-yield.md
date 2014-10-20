---
title: python的yield 用法说明
date: 2014-05-23 16:00:00 +0800
tags:
- yield
---

## python的yield 用法说明

yield 简单说来就是一个生成器，生成器是这样一个函数，它记住上一次返回时在函数体中的位置。对生成器函数的第二次（或第 n 次）调用跳转至该函数中间，而上次调用的所有局部变量都保持不变。

生成器是一个函数， 函数的所有参数都会保留。

第二次调用 此函数时， 使用的参数是前一次保留下的。生成器不仅“记住”了它数据状态。，生成器还“记住”了它在流控制构造（在命令式编程中，这种构造不只是数据值）中的位置。


**yield 生成器的运行机制**

当你向生成器要一个数时，生成器会执行，直至出现 yield 语句，生成器把 yield 的参数给你，之后生成器就不会往下继续运行。 当你向它要下一个数时，它会从上次的状态开始运行，直至出现yield语句，把参数给你，之后停下。如此反复直至退出函数。





***Python 排列，组合生成器***

	def perm(items, n=None):
	    if n is None:
	        n = len(items)
	    for i in range(len(items)):
	        v = items[i:i+1]
	        if n == 1:
	            yield v
	        else:
	            rest = items[:i] + items[i+1:]
	            for p in perm(rest, n-1):
	                yield v + p

***生成组合***

	def comb(items, n=None):
	    if n is None:
	        n = len(items)
	    for i in range(len(items)):
	        v = items[i:i+1]
	        if n == 1:
	            yield v
	        else:
	            rest = items[i+1:]
	            for c in comb(rest, n-1):
	                yield v + c
	
	a = perm('abc')
	for b in a:
	    print b
	    break
	print '-'*20
	for b in a:
	    print b

结果如下：

	102 pvopf006 ~/test> ./generator.py
	abc
	--------------------
	acb
	bac
	bca
	cab
	cba

可以看到，在第一个循环break后，生成器没有继续执行，而第二个循环接着第一个循环执行

***如何判断一个函数是否是一个特殊的 generator 函数？***

可以利用 isgeneratorfunction 判断：


	 >>> from inspect import isgeneratorfunction
	 >>> isgeneratorfunction(fab)
	 True

***return 的作用***

在一个 generator function 中，如果没有 return，则默认执行至函数完毕，如果在执行过程中 return，则直接抛出 StopIteration 终止迭代。

***通过yield来读取文件内容***

另一个 yield 的例子来源于文件读取。如果直接对文件对象调用 read() 方法，会导致不可预测的内存占用。好的方法是利用固定长度的缓冲区来不断读取文件内容。通过 yield，我们不再需要编写读文件的迭代类，就可以轻松实现文件读取：

	def read_file(fpath):
	   BLOCK_SIZE = 1024
	   with open(fpath, 'rb') as f:
	       while True:
	           block = f.read(BLOCK_SIZE)
	           if block:
	               yield block
	           else:
	               return
