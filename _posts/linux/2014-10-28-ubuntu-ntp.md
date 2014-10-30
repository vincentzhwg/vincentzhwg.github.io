---
title: ubuntu时间设置与ntp同步
date: 2014-10-28 16:50:00 +0800
tags:
- ubuntu
- ntp
---

* toc
{:toc}

多台服务器的时候，在一些集群服务上，对于各机器的时间是否一致，有些服务是比较敏感的，使用外网ntp进行时间同步不太可靠，一来可能没有网络，二来外网同步操作有时较慢，所以很有必要自己搭建一个ntp服务。

# 时间设置

## 修改时间

有很多种修改时间的方式，就记一种比较简单有用的就好了，用 -s 参数，后面接时间字符串，如下例：

    sudo date -s "2014-10-30 12:00:00"

## 时区设置

时间同步之前，先设置好时区。这里以设置中国时区为例，将`Asia/Shanghai`这一行内容放到`/etc/timezone`文件中，命令如下：

    sudo echo "Asia/Shanghai" > /etc/timezone





# ntp

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

## ntp同步搭建实例

### 主服务器搭建

在内网选定一机器，运行以下命令安装ntp服务

    sudo apt-get install ntp

然后用以下内容替换掉`/etc/ntp.conf`原来的内容，其中对内网网段的对时请求的配置根据需要进行修改：

    # /etc/ntp.conf, configuration for ntpd; see ntp.conf(5) for help
    
    driftfile /var/lib/ntp/ntp.drift #草稿文件
    
    #日志文件
    statsdir /var/log/ntpstats/
    
    statistics loopstats peerstats clockstats
    filegen loopstats file loopstats type day enable
    filegen peerstats file peerstats type day enable
    filegen clockstats file clockstats type day enable
    
    #上层ntp server
    server 2.cn.pool.ntp.org prefer
    server 0.asia.pool.ntp.org prefer
    server 3.asia.pool.ntp.org prefer
    
    #让NTP Server和其自身保持同步，如果在/etc/ntp.conf中定义的server都不可用时，将使用local时间作为ntp服务提供给ntp客户端
    server 127.127.1.0
    fudge 127.127.1.0 stratum 5
    
    #不允许来自公网上ipv4和ipv6客户端的访问
    restrict -4 default kod notrap nomodify nopeer noquery 
    restrict -6 default kod notrap nomodify nopeer noquery
    
    #允许内网网段的对时请求
    restrict 192.168.1.0 mask 255.255.255.0 nomodify notrap
    
    # Local users may interrogate the ntp server more closely.
    restrict 127.0.0.1
    restrict ::1


### 内网剩余机器的对时

内网剩余机器的对时有两种方式：ntpdate 或 ntpd 。

#### ntpdate方式

**使用ntpdate方式，同步有些粗暴，是把时间直接置为主服务器的时间，有可能对某些对时间敏感的服务有影响。**

比较简单的就是使用ntpdate命令方式，配合crontab进行定期对时。假设上面主服务器的IP地址为192.168.1.123，那么可以在剩余的每一台机器的root用户的crontab中添加如下语句：

    05 */1 * * * ntpdate 192.168.1.123

这样即可每小时的05分进行一次时间同步。


#### ntpd方式

**nptd方式，当发觉本机与同步服务器的时间不一致时，不会马上置为同步服务器的时间，而是通过加快或变慢来逐步与同步服务器的时间来保持一致。**

ntpd方式，要求在需要同步服务的机器上也需要安装ntp，但只用于与主服务器进行同步，不对其他机器提供同步服务。这里仍然假设上面安装好ntp同步服务的主服务器的IP地址为192.168.1.123。

在内网剩余的每一台机器上安装ntp:

    sudo apt-get install ntp

使用以下内容配置`/etc/ntp.conf`，其中同步主服务器的IP地址根据你自己的需要进行修改：

    # /etc/ntp.conf, configuration for ntpd; see ntp.conf(5) for help
    
    driftfile /var/lib/ntp/ntp.drift #草稿文件
    
    #日志文件
    statsdir /var/log/ntpstats/
    
    statistics loopstats peerstats clockstats
    filegen loopstats file loopstats type day enable
    filegen peerstats file peerstats type day enable
    filegen clockstats file clockstats type day enable
    
    
    #让NTP Server为内网的ntp服务器
    server 192.168.1.123
    fudge 192.168.1.123     stratum 5
    
    #不允许来自公网上ipv4和ipv6客户端的访问
    restrict -4 default kod notrap nomodify nopeer noquery 
    restrict -6 default kod notrap nomodify nopeer noquery
    
    # Local users may interrogate the ntp server more closely.
    restrict 127.0.0.1
    restrict ::1

    

ntpd方式，使用时要求本机与同步服务器的时间差距不能大于**1000**秒，所以一开始的时间，可以把本机粗略设置个与同步服务器相近的时间，后面通过ntpd就会同步成一致了。


