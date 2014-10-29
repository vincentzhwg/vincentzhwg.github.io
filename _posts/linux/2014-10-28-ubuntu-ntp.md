---
title: ubuntu搭建ntp时间同步服务
date: 2014-10-28 16:50:00 +0800
published: false
tags:
- ubuntu
- ntp
---

* toc
{:toc}

多台服务器的时候，在一些集群服务上，对于各机器的时间是否一致，有些服务是比较敏感的，使用外网ntp进行时间同步不太可靠，一来可能没有网络，二来外网同步操作有时较慢，所以很有必要自己搭建一个ntp服务。





## 安装ntp

在服务器内网中选取一台机器做为ntp服务器，运行安装命令

    sudo apt-get install ntp

## 配置选项说明

ntp的配置文件为`/etc/ntp.conf`，里面有好多配置选项，下面进行一下说明。

### server

利用server设定上层NTP服务器，上层NTP服务器的设定方式为：  

    server    [IP OR HOSTNAME]    [PREFER]  [DRIFTFILE]
  
参数说明：  

- ip or hostname ：上层ntp服务器的ip地址或者域名  
- prefer         ： 表示优先使用的主机  
- driftfile      ：记录时间差异。因为预设的NTP Server本身的时间计算是依据BIOS的晶片震荡周期频率来计算的，但是这个数值与上层Time Server有误差。所以NTP这个daemon会自动的去计算我们自己主机的频率与上层TimeServer的频率，并且将两个频率的误差记录下来，记录下来的档案就是在driftfile后面接的完整文件名中了

### restrict

    restrict [ip] mask [netmask_ip] [parameter]
  
其中 parameter 可用参数如下，多个参数之间用空格隔开即可：  

- ignore      ：   拒绝所有类型的ntp连接  
- nomodify    ：   客户端不能使用ntpc与ntpq两支程式来修改服务器的时间参数  
- noquery     ：   客户端不能使用ntpq、ntpc等指令来查询服务器时间，等于不提供ntp的网络校时  
- notrap      ：   不提供trap这个远程时间登录的功能  
- notrust     ：   拒绝没有认证的客户端  
- nopeer      ：   不与其他同一层的ntp服务器进行时间同步 


## 服务运行情况查看

### ntpq

可运行以下命令查看ntp服务运行情况：

    ntpq -p

会得到类似下面的输出结果：

<img src="/images/linux/ntp-q.png"/>

上图中参数含义如下：

- remote  ：   本地机器所连接的远程NTP服务器。每一个remote地址前有一个符号，含义如下：
    * \* 它告诉我们远端的服务器已经被确认为我们的主NTP Server，我们系统的时间将由这台机器所提供  
    * \+ 它将作为辅助的NTP Server和带有 \* 号的服务器一起为我们提供同步服务.当 \* 号服务器不可用时它就可以接管  
    * \- 远程服务器被clustering algorithm认为是不合格的NTP Server  
    * x  远程服务器不可用 
- refid   ：   指的是参考的上一层NTP主机的地址  
- st      ：   远程服务器的级别。由于NTP是层型结构，有顶端的服务器,多层的Relay Server再到客户端.所以服务器从高到低级别可以设定为1-16.为了减缓负荷和网络堵塞,原则上应该避免直接连接到级别为1的服务器的  
- when    ：   用做计时，用来告诉我们还有多久本地机器就需要和远程服务器进行一次时间同步  
- poll    ：   本地主机和远程服务器多少时间进行一次同步（单位为秒）  
- reach   ：   这是一个八进制值，表示已经向上层NTP服务器要求更新的次数。每成功连接一次，它的值就加1  
- delay   ：   网络传输过程中延迟的时间，单位为微秒  
- offset  ：   我们本地机和服务器之间的时间差别。单位为毫秒  
- jitter  ：   Linux系统时间与BIOS硬件时间的差异时间，单位为微秒 
