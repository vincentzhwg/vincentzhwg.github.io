---
title: ubuntu导入网站证书与设置网络代理的方法
date: 2014-06-16 16:00:00 +0800
tags:
- ubuntu
---

## linux下使用网络代理的命令行方式

	export http_proxy=http://ip:port/
	export https_proxy=http://ip:port/
	export ftp_proxy=http://ip:port/
	
## 导入证书的两种方式





#### 第一种：直接追加到系统级，不太推荐，以后不好维护

一个快速增加ubuntu的网站证书的方式，就是在 /etc/ssl/certs/ca-certificates.crt 后面直接追加你手上有的 crt 文件的内容，就这么方便快速。

#### 第二种：通过命令行添加

	## 在 /usr/share/ca-certificates 目录下新建存放证书所对应的网站目录，这里以 github 为例进行说明
	sudo mkdir /usr/share/ca-certificates/github.com/
	
	## 进入到所创建好的目录，这里使用了一种 !!:2 来指代上面的目录，索引从0开始，即 sudo对应0，mkdir对应1
	cd !!:2
	
	## 将准备好的要导入的证书复制到当前目录下
	sudo cp /path/to//github.com.crt .
	
	## 运行配置命令导入，运行此命令后会出现类似下图所示的配置选项，相应配置即可
	sudo dpkg-reconfigure ca-certificates
	
	## 配置好后再更新一下
	sudo update-ca-certificates

配置示例图片

<img src="/images/2014/github_ca_certification.png" alt="" width="747" height="454" />
