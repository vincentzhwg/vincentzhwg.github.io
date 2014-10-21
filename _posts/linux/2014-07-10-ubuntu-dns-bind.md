---
title: ubuntu搭建本地网DNS服务器
date: 2014-07-10 16:00:00 +0800
tags:
- dns
---

在公司的服务器的内网搭建一个本地的DNS服务器,还是有不少好处的.本文是搭建过程的记录.

系统:ubuntu12.04 LTS server

## bind介绍

BIND (Berkeley Internet Name Domain)是Domain Name System (DNS) 协议的一个实现，提供了DNS主要功能的开放实现，包括

- 域名服务器 (named)
- DNS解析库函数
- DNS服务器运行调试所用的工具

Bind是一款开放源码的DNS服务器软件，由美国加州大学Berkeley分校开发和维护的，按照ISC的调查报告，BIND是世界上使用最多最广泛的域名服务系统。

BIND，也是我们常说的named，由于多数网络应用程序使用其功能，所以在很多BIND的弱点及时被发现。主要分为三个版本：

- v4 : 1998年多数UNIX捆绑的是BIND4，已经被多数厂商抛弃了，除了OpenBSD还在使用。OpenBSD核心人为BIND8过于复杂和不安全，所以继续使用BIND4。这样一来BIND8/9的很多优点都不包括在v4中。
- v8 : 就是如今使用最多最广的版本，其详细内容可以参阅 BIND 8+ 域名服务器安全增强
- v9 : 最新版本的BIND，全部重新写过，免费（但是由商业公司资助），也添加了许多新的功能（但是安全上也可能有更多的问题）。BIND9在2000年十月份推出，现在稳定版本是9.3.2








## bind9的安装

	sudo apt-get install bind9

	## 辅助工具及文档,可选安装
	sudo apt-get install bind9-host dnsutils bind9-doc


## 配置

**SOA配置项解释**

在配置之前，对重要的SOA配置项先做下说明。

Bind的SOA记录：每个Zone仅有一个SOA记录。SOA记录包括Zone的名字，一个技术联系人和各种不同的超时值。

超时值的说明如下：

- Serial : 数值Serial代表这个Zone的序列号。作用：供Slave DNS判断是否从Master DNS获取新数据。每次Zone文件更新，都需要修改Serial数值。RFC1912 2.2建议的格式为YYYYMMDDnn 其中nn为修订号，采用每次自增1即可
- Refresh : 数值Refresh设置Slave DNS多长时间与Master Server进行Serial核对。目前Bind的notify参数可设置每次Master DNS更新都会主动通知Slave DNS更新，Refresh参数主要用于notify参数关闭时
- Retry : 数值Retry设置当Slave DNS试图获取Master DNS Serial时，如果Master DNS未响应，多长时间重新进行检查
- Expire : 数值Expire将决定Slave DNS在没有Master DNS的情况下权威地提供域名解析服务的时间长短
- Negative Cache TTL : 在8.2版本之前，由于没有独立的 $TTL 指令，所以通过 SOA 最后一个字段来实现。但由于 BIND 8.2 后出现了 $TTL 指令，该部分功能就不再由 SOA 的最后一个字段来负责，由 $TTL 全权负责，SOA 的最后一个字段专门负责 negative answer ttl（negative caching）。该项用于配置其他服务器对于由DNS服务器返回的no-such-domain(NXDOMAIN)响应的缓存时间。

#### 主服务器的配置

这里还未加上一些权限控制及chroot设置,若想加强安全考虑,可加上这些配置.

这里假设安装dns服务机器的IP地址为 192.168.1.10

---------------
**/etc/bind/named.conf.options**

	options {
	    directory "/var/cache/bind";
	
	    //当本dns服务器无法解析时,转发解析请求,这里配一些外面常用的dns地址即可
	    forwarders {
	        114.114.114.114;
	        114.114.115.115;
	    };
	
	    dnssec-validation auto;
	    auth-nxdomain no;    # conform to RFC1035
	    listen-on-v6 { any; };
	};

------------

**/etc/bind/named.conf.local**

	// 这里以配置 example.com 域名进行举例
	zone "example.com" {
	    type master;
	    file "/etc/bind/db.example.com";
	};
	
	// 反向解析配置, IP地址需要倒过来写, 如 192.168.X.X 网段,得写成如下面的 169.192.in-addr.arpa
	zone "168.192.in-addr.arpa" {
	    type master;
	    file "/etc/bind/db.168.192";
	};

---------------

**/etc/bind/db.example.com**

	$TTL    604800
	@   IN  SOA ns1.example.com. root.example.com. (   ; 这里假设将本机的域名定义 ns1
	            2014071101      ; Serial  序号,建议每次更新后加1或用当前时间值,只要大于原来的数值即可
	            2H              ; Refresh 刷新时间
	            4M              ; Retry   失败时再次重试的时间间隔
	            7D              ; Expire  缓存时间
	            1D )            ; Negative Cache TTL
	;
	@   IN  NS  ns1.example.com.    ; 最后一定要有个 . 符号
	
	ns1 IN A 192.168.1.10   ;定义ns1的对应关系
	h01 IN A 192.168.0.10   ;定义h01的对应关系

----------------

**/etc/bind/db.168.192**

	$TTL    604800
	@   IN  SOA ns1.example.com. root.example.com. (   ; 这里假设将本机的域名定义 ns1
	            2014071101      ; Serial  序号,建议每次更新后加1或用当前时间值,只要大于原来的数值即可
	            2H              ; Refresh 刷新时间
	            4M              ; Retry   失败时再次重试的时间间隔
	            7D              ; Expire  缓存时间
	            1D )            ; Negative Cache TTL
	;
	@   IN  NS  ns1.example.com.
	
	10.1  IN PTR ns1.example.com.  ; 反过来写IP地址,且最后一定要有个 . 符号,这一行表示反向解析 192.168.1.10为ns1.example.com
	10.0  IN PTR h01.example.com.

-----------------

#### 配置项的检查

	## 定义了那么多配置项,万一写错了语法可怎么办,有一个命令可进行配置项的检查
	named-checkconf
	## 运行此命令,如果没有错误则没有任何输出,若有错误会输出错误提示

#### 启动,停止

	## 启动
	sudo service bind9 start
	
	## 停止
	sudo service bind9 stop
	
	## 重启
	sudo service bind9 restart

#### 对各机器的hostname的配置

当配好dns服务器后，充分利用dns服务器的优势，对各机器的hostname建议做以下配置。
假设，现有一台机器，已将dns配置指向了搭建好的dns服务器，而在dns服务器上为该机的IP地址所分配的域名为 a01.example.com ,那么该机的hostname可做如下配置：

	### 下面命令用于更改hostname，只要不重启就是一直生效的，对所有登录用户都生效。运行该命令时，第一次运行可能报一个无法解析的错误，不用理，再运行一次即好。
	sudo hostname a01\.example\.com

	### 就算重启之后还是同样生效
	sudo echo 'a01.example.com' > /etc/hostname

这样配置之后，一两分钟后就会生效，有一个方便性，就是可直接使用 a01 来代替 a01.example.com 了，即可直接 ping a01 或 ssh a01 ，而无需再去动 /etc/hosts 文件了，相当方便。

## 增加从服务器

从服务器不是必须的,但增加一台从服务器,可加强容灾能力.从服务器不一定要在同一个网段,但至少要能互相访问.

这里假设主服务器就采用上面配置好的192.168.1.10,而从服务器我们选定机器的IP地址为192.168.1.11,需要做如下主从服务器的配置与更改.

**主服务器的/etc/bind/named.conf.local**

	// 这里以配置 example.com 域名进行举例
	zone "example.com" {
	    type master;
	    file "/etc/bind/db.example.com";
	    allow-transfer { 192.168.1.11 };     // 只传输给从服务器,增加安全性
	    notify yes;                          // 发出通知
	    also-notify { 192.168.1.11 };        // 通知从服务器
	};
	
	// 反向解析配置, IP地址需要倒过来写, 如 192.168.X.X 网段,得写成如下面的 169.192.in-addr.arpa
	zone "168.192.in-addr.arpa" {
	    type master;
	    file "/etc/bind/db.168.192";
	    allow-transfer { 192.168.1.11 };
	    notify yes;
	    also-notify { 192.168.1.11 };
	};

--------------

**主服务器的/etc/bind/db.example.com**

	$TTL    604800
	@   IN  SOA ns1.example.com. root.example.com. (   ; 这里假设将本机的域名定义 ns1
	            2014071102      ; Serial  序号,建议每次更新后加1或用当前时间值,只要大于原来的数值即可
	            12H             ; Refresh 刷新时间
	            4M              ; Retry   失败时再次重试的时间间隔
	            7D              ; Expire  缓存时间
	            1D )            ; Negative Cache TTL
	;
	@   IN  NS  ns1.example.com.    ; 最后一定要有个 . 符号
	@   IN  NS  ns2.example.com.
	
	ns1 IN A 192.168.1.10   ;定义ns1的对应关系
	ns2 IN A 192.168.1.11   ;定义ns2的对应关系
	h01 IN A 192.168.0.10   ;定义h01的对应关系

--------------

**主服务器的/etc/bind/db.168.192**

	$TTL    604800
	@   IN  SOA ns1.example.com. root.example.com. (   ; 这里假设将本机的域名定义 ns1
	            2014071102      ; Serial  序号,建议每次更新后加1或用当前时间值,只要大于原来的数值即可
	            12H             ; Refresh 刷新时间
	            4M              ; Retry   失败时再次重试的时间间隔
	            7D              ; Expire  缓存时间
	            1D )            ; Negative Cache TTL
	;
	@   IN  NS  ns1.example.com.
	@   IN  NS  ns2.example.com.
	
	10.1  IN PTR ns1.example.com.  ; 反过来写IP地址,且最后一定要有个 . 符号,这一行表示反向解析 192.168.1.10为ns1.example.com
	11.1  IN PTR ns2.example.com.
	10.0  IN PTR h01.example.com.

------------------

**从服务器的/etc/bind/named.conf.options**

	options {
	    directory "/var/cache/bind";
	
	    //当本dns服务器无法解析时,转发解析请求,这里配一些外面常用的dns地址即可
	    forwarders {
	        114.114.114.114;
	        114.114.115.115;
	    };
	
	    dnssec-validation auto;
	    auth-nxdomain no;    # conform to RFC1035
	    listen-on-v6 { any; };
	};

----------------------

**从服务器的named.conf.local**

	zone "example.com" {
	    type slave;                               //设为从服务器
	    masters { 192.168.1.10; };                //指向主服务器
	    file "/var/cache/bind/db.example.com";    //设置主服务器同步过来的设置文件的本机存放路径,这里建议使用 /var/cache/bind 目录,以避免同步时文件读写权限问题
	    allow-transfer { none; };                 //设置从服务器不传输内容给其余服务器了
	};
	
	zone "168.192.in-addr.arpa" {
	    type slave;
	    masters { 192.168.1.10; };
	    file "/var/cache/bind/db.168.192";
	    allow-transfer { none; };
	};
