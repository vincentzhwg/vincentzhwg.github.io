---
title: golang通过thrift操作Hbase
date: 2014-08-04 16:00:00 +0800
tags:
- hbase
- golang
---

hbase的原生操作语言是java，不过线上一个系统中使用了go语言，在网上找了好多资料，看来通过thrift接口对于go来说是比较好的一种方式，下面是记录如何通过thrift接口让go语言操作hbase的数据。




## 准备工作

golang开发环境及Hbase都已经搭建好

## 编译thrift

	### 通过go get方式下载thrift的源码
	go get git.apache.org/thrift.git/lib/go/thrift
	
	## 进入到thrift目录
	cd /path/to/git.apache.org/thrift.git/lib/go/thrift
	
	## 切换成相应所要用的分支，写本文的时候，我选取的分支为 0.9.2，可通过 git branch -a 命令查看，选择你想要的一个分支
	## 之所以多这一个步骤，是因为在master分支时编译时报了个编译错误，我就切换分支之后就没有错了，那就切换吧
	## 如果你在下面编译中不会出错，可不考虑切换分支
	git checkout --track origin/0.9.2
	
	## 本文使用系统环境为 ubuntu12.04 LTS server版，所以按照官方文档，先执行以下命令预安装一些包
	## 如果非 ubuntu/debian 系统，请忽略以下命令，并在编译时找办法解决库包的依赖问题
	sudo apt-get install libboost-dev libboost-test-dev libboost-program-options-dev libevent-dev automake libtool flex bison pkg-config g++ libssl-dev
	
	## 编译  请注意，不同分支的编译方法不一样，我这里说的是0.9.2的编译方法，具体编译方法可查看 compliler/cpp/REAMDE.md 文件，可能有些分支没有
	## 编译过程中，如有什么错误，缺库缺包啥的，请自行解决
	cd compiler/cpp
	mkdir build
	cd build
	cmake ..
	make
	
	## 编译完成之后，就会在 build 目录下生成 thrift 可执行文件，试着执行下，看下是否编译安装成功
	## 成功情况下可看到输出的帮助信息
	./thrift -help
	
	## 最后可将该执行文件拷贝到你想要的地方，方便使用
	mv ./thrift /path/to/you/want

## 编译Hbase的thrift文件


	### 从hbase官网下载所需要的Hbase版本的源代码，记住是源代码，而不是已经封装好的版本
	### 解压打开源代码
	tar zxvf hbase-<version>-src.tar.gz
	cd hbase-<version>
	
	### 执行find命令找出 Hbase.thrift 文件位置
	find . -name "Hbase.thrift"
	## 得到find命令的输出为  ./hbase-thrift/src/main/resources/org/apache/hadoop/hbase/thrift/Hbase.thrift
	## 拷贝到与 thrift 相同目录下，方便后面的操作
	cp ./hbase-thrift/src/main/resources/org/apache/hadoop/hbase/thrift/Hbase.thrift /path/to/where/thrift/dir/path
	
	## 通过以上步骤之后，那么前面所要的 thrift 可执行文件 与 Hbase.thrift 文件都已经在我们设定好的同一个目录下了
	## 接下来，进行编译 Hbase的thrift的操作
	## 先进入到 thrift 与 Hbase.thrift 的目录
	cd /path/to/where/thrift/dir/path
	
	## 编译go版本的Hbase的thrift
	./thrift -r --gen go Hbase.thrift
	
	## 运行命令后，在目录下将新生成一名为 gen-go 的目录，里面就包含了所需的go文件了
	## gen-go的目录结构如下：
	## gen-go
	##   └── hbase
	##       ├── constants.go
	##       ├── hbase.go
	##       ├── hbase-remote         ## 该文件夹是demo文件夹，可以删掉
	##       │   └── hbase-remote.go  ## 客户端的测试代码，直接可拿来用，可做为client端的代码参考，使用时记得修改下hbase包的引用路径
	##       └── ttypes.go
	
	## 现在通过 thrift 给go语言自动化生成的Hbase的thrift相关文件还不够完善，直接使用会出错，需要做出一些修改
	
	## 使用编辑 hbase.go 文件，并执行全局替换命令
	## vim gen-go/hbase/hbase.go
	## 然后执行修改命令
	## %s#\(_key\d\+\) = temp#\1 = string(temp)#g
	## 保存退出
	
	## 同样的用vim编辑 ttypes.go 文件
	## vim gen-go/hbase/ttypes.go
	## 然后执行修改命令
	## %s#\(_key\d\+\) = temp#\1 = string(temp)#g
	## 保存退出


## thrift调用测试

本例中的测试使用 thrift 当时所生成的测试代码


	## 将上面编译好的 hbase 的相关代码放到相应位置，这里路径为 $GOPATH/src/localhost
	mkdir -p $GOPATH/src/localhost
	cp -r /path/to/gen-go/hbase $GOPATH/src/localhost/
	
	## 新建测试代码的目录，使用 thrift 生成的 demo 代码
	mkdir -p $GOPATH/src/localhost/test/hbaseThriftTest
	cp -r /path/to/gen-go/hbase/hbase-remote/hbase-remote.go $GOPATH/src/localhost/test/hbaseThriftTest/main.go
	
	## 编辑修改 main.go 文件
	cd $GOPATH/src/localhost/test/hbaseThriftTest
	## vim main.go
	## 将其中的 import 里面的 "hbase" 改成 "localhost/hbase" 即可，保存退出
	
	## 编译
	go install
	## 一切顺利的话，这里就在 $GOPATH/bin/hbaseThriftTest 的可执行文件，记得把 $GOPATH/bin 加入到 $PATH 变量中
	
	## 测试运行一下
	## 查看帮助命令
	hbaseThriftTest
	
	## 测试表是否 enabled , 这里测试的表名为 t1
	hbaseThriftTest -h host:port isTableEnabled 't1'
	
	## 获取t1表的r1行的所有值
	hbaseThriftTest -h host:port getRow 't1' 'r1' 'null'


## hbase的thrift2接口

后面在使用过程中，发觉原来hbase有两套thrift接口，还有一套thrift2的接口，这套接口更接近于java的API用法，且thrift2版所生成的golang的代码，不需要任何修改就可以使用了，推荐使用下。

启动hbase的thrift2服务

	### 启动thrift2之前，记得先关掉原来的thrift服务，或者在启动thrift2服务时指定别的端口什么的，以免冲突
	
	cd /path/to/hbase
	bin/hbase-daemon.sh start thrift2

## thrift2的编译


	### 解压打开源代码
	tar zxvf hbase-<version>-src.tar.gz
	cd hbase-<version>
	
	### 执行find命令找出 Hbase.thrift 文件位置，这里与上面不同的只是 Hbase.thrift 换成了 hbase.thrift 而已
	find . -name "hbase.thrift"
	## 得到find命令的输出为  ./hbase-thrift/src/main/resources/org/apache/hadoop/hbase/thrift2/hbase.thrift
	
	## 拷贝到与 thrift 相同目录下，方便后面的操作
	cp ./hbase-thrift/src/main/resources/org/apache/hadoop/hbase/thrift2/hbase.thrift /path/to/where/thrift/dir/path
	
	## 通过以上步骤之后，那么前面所要的 thrift 可执行文件 与 hbase.thrift 文件都已经在我们设定好的同一个目录下了
	## 接下来，进行编译 Hbase的thrift的操作
	## 先进入到 thrift 与 Hbase.thrift 的目录
	cd /path/to/where/thrift/dir/path
	
	## 编译go版本的Hbase的thrift
	./thrift -r --gen go hbase.thrift
	
	## 运行命令后，在目录下将新生成一名为 gen-go 的目录，里面就包含了所需的go文件了
	## gen-go的目录结构如下：
	## gen-go
	##   └── hbase2
	##       ├── constants.go
	##       ├── thbaseservice.go
	##       ├── t_h_base_service-remote         ## 该文件夹是demo文件夹，可以删掉
	##       │   └── t_h_base_service-remote.go  ## 客户端的测试代码，直接可拿来用，可做为client端的代码参考，使用时记得修改下hbase包的引用路径
	##       └── ttypes.go
	
	## 把hbase2文件夹拷贝到你的golang开发环境中，就可以直接使用了
