---
title: python的with用法
date: 2014-05-23 17:00:00 +0800
tags:
- with
---


## With语句是什么?

有一些任务，可能事先需要设置，事后做清理工作。对于这种场景，Python的with语句提供了一种非常方便的处理方式。一个很好的例子是文件处理，你需要获取一个文件句柄，从文件中读取数据，然后关闭文件句柄。

如果不用with语句，代码如下：

	file = open("/tmp/foo.txt")
	data = file.read()
	file.close()

这里有两个问题:

1. 一是可能忘记关闭文件句柄；
2. 二是文件读取数据发生异常，没有进行任何处理。






下面是处理异常的加强版本：

	try:
	    f = open('xxx')
	except:
	    print 'fail to open'
	    exit(-1)
	try:
	    do something
	except:
	    do something
	finally:
	     f.close()

虽然这段代码运行良好，但是太冗长了。

这时候就是with一展身手的时候了。**除了有更优雅的语法，with还可以很好的处理上下文环境产生的异常。**

下面是with版本的代码：

	with open("/tmp/foo.txt") as file:
	    data = file.read()

## with如何工作?

- 紧跟with后面的语句被求值后，返回对象的 \_\_enter\_\_() 方法被调用，这个方法的返回值将被赋值给as后面的变量。
- 当with后面的代码块全部被执行完之后，将调用前面返回对象的 \_\_exit\_\_()方法。

下面例子可以具体说明with如何工作：

	#!/usr/bin/env python
	# with_example01.py
	
	class Sample:
	    def __enter__(self):
	        print "In __enter__()"
	        return "Foo"
	
	    def __exit__(self, type, value, trace):
	        print "In __exit__()"
	
	def get_sample():
	    return Sample()
	
	with get_sample() as sample:
	    print "sample:", sample

运行代码，输出如下


	bash-3.2$ ./with_example01.py
	In __enter__()
	sample: Foo
	In __exit__()

正如你看到的:
1.  \_\_enter\_\_()方法被执行
2.  \_\_enter\_\_()方法返回的值 - 这个例子中是"Foo"，赋值给变量'sample'
3.  执行代码块，打印变量"sample"的值为 "Foo"
4.  \_\_exit\_\_()方法被调用 with真正强大之处是它可以处理异常。

可能你已经注意到Sample类的 \_\_exit\_\_ 方法有三个参数 val, type 和 trace。 这些参数在异常处理中相当有用。我们来改一下代码，看看具体如何工作的。

	#!/usr/bin/env python
	# with_example02.py
	
	class Sample:
	    def __enter__(self):
	        return self
	
	    def __exit__(self, type, value, trace):
	        print "type:", type
	        print "value:", value
	        print "trace:", trace
	
	    def do_something(self):
	        bar = 1/0
	        return bar + 10
	
	with Sample() as sample:
	    sample.do_something()

这个例子中，with后面的get_sample()变成了Sample()。这没有任何关系，只要紧跟with后面的语句所返回的对象有 \_\_enter\_\_() 和 \_\_exit\_\_() 方法即可。此例中，Sample()的 \_\_enter\_\_() 方法返回新创建的Sample对象，并赋值给变量sample。 

代码执行后：

	bash-3.2$ ./with_example02.py
	type: <type 'exceptions.ZeroDivisionError'>
	value: integer division or modulo by zero
	trace: <traceback object at 0x1004a8128>
	Traceback (most recent call last):
	  File "./with_example02.py", line 19, in <module>
	    sample.do_something()
	  File "./with_example02.py", line 15, in do_something
	    bar = 1/0
	ZeroDivisionError: integer division or modulo by zero


实际上，在with后面的代码块抛出任何异常时，\_\_exit\_\_() 方法被执行。正如例子所示，异常抛出时，与之关联的type，value和stack trace传给 \_\_exit\_\_() 方法，因此抛出的ZeroDivisionError异常被打印出来了。开发库时，清理资源，关闭文件等等操作，都可以放在 \_\_exit\_\_ 方法当中。

另外，\_\_exit\_\_ 除了用于tear things down，还可以进行异常的监控和处理，注意后几个参数。要跳过一个异常，只需要返回该函数True即可。

下面的样例代码跳过了所有的TypeError，而让其他异常正常抛出。

	def __exit__(self, type, value, traceback):
	    return isinstance(value, TypeError)

上文说了 \_\_exit\_\_ 函数可以进行部分异常的处理，如果我们不在这个函数中处理异常，他会正常抛出，这时候我们可以这样写（python 2.7及以上版本，之前的版本参考使用contextlib.nested这个库函数）：

	try:
	    with open( "a.txt" ) as f :
	        do something
	except xxxError:
	    do something about exception

总之，with-as表达式极大的简化了每次写finally的工作，这对保持代码的优雅性是有极大帮助的。

如果有多个项，我们可以这么写：

	with open("x.txt") as f1, open('xxx.txt') as f2:
	    do something with f1,f2

因此，Python的with语句是提供一个有效的机制，让代码更简练，同时在异常产生时，清理工作更简单。
