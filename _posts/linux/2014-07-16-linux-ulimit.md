---
title: linux服务器调整ulimit数值
date: 2014-07-16 16:00:00 +0800
tags:
- ulimit
---

系统：ubuntu 12.04 LTS server

今天，在linux服务器上碰到打开文件句柄数限制，就想起要更改ulimit的数值了，不过无论是在命令中运行 ulimit -n 65536 或在 ~/.bashrc 中添加这一行均出错或无法达到目的。

后来找到如下方式可行，记录如下：

假设工作用户为 work ,首先需要编辑文件

	sudo vim /etc/security/limits.conf


在文件的最后面添加如下内容

	## 设置root帐户
	## nofile 表示文件句柄数的设置
	## soft即是软限制，hard是硬限制。用户可以超过soft设置的值，但一定不能超过hard 的值。一般soft比hard小
	root soft nofile 65536
	root hard nofile 65536
	
	## 设置work帐户
	work soft nofile 65536
	work hard nofile 65536

如果不想一个个用户设置的话，那可以设置全局，如下

	## 全局设置
	## nofile 表示文件句柄数的设置
	## soft即是软限制，hard是硬限制。用户可以超过soft设置的值，但一定不能超过hard 的值。一般soft比hard小
	* soft nofile 65536
	* hard nofile 65536

这样设置之后，退出重新登录即生效。

若想对work帐户的nofile再做更进一步的设置，可在 ~/.bashrc 文件下添加一行，如

	ulimit -n 51200

即设置文件句柄数限制为51200。注意，在 ~/.bashrc 中的数值不可超过 hard 所设置的值。
