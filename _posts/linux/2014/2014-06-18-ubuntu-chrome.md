---
title: ubuntu下chrome标签乱码解决办法
date: 2014-06-18 17:00:00 +0800
tags:
- ubuntu
---

#### ubuntu下chrome标签乱码解决办法

直接替换 /etc/fonts/conf.d/49-sansserif.conf 的文件内容如下:

	<?xml version="1.0"?>
	<!DOCTYPE fontconfig SYSTEM "fonts.dtd">
	<fontconfig>
	<!--
	  If the font still has no generic name, add sans-serif
	 -->
		<match target="pattern">
			<test qual="all" name="family" compare="not_eq">
				<string>WenQuanYi Micro Hei</string>
			</test>
			<test qual="all" name="family" compare="not_eq">
				<string>WenQuanYi Micro Hei</string>
			</test>
			<test qual="all" name="family" compare="not_eq">
				<string>WenQuanYi Micro Hei Mono</string>
			</test>
			<edit name="family" mode="append_last">
				<string>WenQuanYi Micro Hei</string>
			</edit>
		</match>
	</fontconfig>
