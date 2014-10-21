---
title: linux中命令的复用
dae: 2014-07-17 16:00:00 +0800
tags:
- linux
---

在linux终端下成天地敲命令，不使用一些偷懒的方法，敲键盘都累呀。有以下敲命令的偷懒方式：

- 根据关键字快速查找命令，按 Ctrl + r 组合键，会出现 (reverse-i-search)`': 的提示，这时输入想要的命令关键字，会出现以前输入过跟关键字相同的历史命令，这时再按 Ctrl + r 组合键，即可翻阅，待找到想要的那一条命令时，按回车是直接执行，按tab键则是不执行，但会把该条命令显示在终端上即可进行修改后再执行。
- 直接输入 !! ，按回车就会执行刚刚执行过的命令。
- 命令有时有很多参数及选项，但有时经常想复用命令，但只是想变动下其中的一些参数。那么可以使用 !!:n ，其中的 n 为数字，代表刚执行过的命令中的第几项，从0开始数起。示例如下：

>	## 创建一目录
	mkdir /tmp/abc
	## 创建完后，进入到该目录
	cd ::!1

- 使用 !cmd 模式，其中cmd为命令关键字，那么就会执行跟该关键字匹配的最后执行的那条命令。示例如下：


>	cp /tmp/a /tmp/b
	cp /tmp/c /tmp/d
	
>	!cp
	## 上面的 !cp 将会执行最近执行过的 cp /tmp/c /tmp/d
	
>	cp /tmp/e /tmp/f
	
>	!cp
	## 上面的 !cp ,这一次将会执行最近执行过的 cp /tmp/e /tmp/f