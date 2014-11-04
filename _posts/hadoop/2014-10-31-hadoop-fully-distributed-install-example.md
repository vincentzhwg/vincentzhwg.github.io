---
title: hadoop--完全分布式安装实例
date: 2014-10-31 10:40:00 +0800
tags:
- hadoop
- hdfs
- install
---

* toc
{:toc}

# 机器环境

- 3台处于同一网段的虚拟机，IP地址分别为 192.168.1.171, 192.168.1.189, 192.168.1.196
- 所有机器节点均能访问外网
- 系统均为 ubuntu12.04 LTS server x86-64
- 所有机器节点的登录帐户为 work ，但密码并不一定一致
- 所要安装的hadoop版本为stable版：2.5.1





# 每个节点的环境准备

## 机器帐户及基准目录

所有节点均使用work帐户，即$HOME路径为 /home/work ，以后所有的资源及文件均会在 $HOME 目录下的 VMBigData 目录下，即后面所说的 **VMBigData** 目录均指 /home/work/VMBigData 。

在 VMBigData 下建立 resource 目录，存放所有原始资源文件，如jdk安装文件，hadoop安装包等。

## 安装ssh, rsync

使用的是ubuntu，又可以访问外网，所以使用以下命令完成安装：

- sudo apt-get install ssh
- sudo apt-get install rsync


## hostname及域名配置

如果允许的话，还是尽量在内网中搭建自己的DNS服务，搭建可参考[ubuntu搭建本地网DNS服务器](/2014/07/10/ubuntu-dns-bind.html){:target="_blank"}。

若是不能自己搭建DNS服务的话，那就使用各机器的 `/etc/hosts` 来指定IP地址与域名的关系。

本实例中对3台
- h171.vm.com : 192.168.1.171
- h189.vm.com : 192.168.1.189
- h196.vm.com : 192.168.1.196

在后面的记录中，均使用节点简称来指某一个节点，如h171指h171.vm.com，h189指h189.vm.com，h196指h196.vm.com。

## 所有节点的时间同步

hadoop对于时间还是有一些要求，每一个节点的时间需要一致，所以最好搭建一个ntp服务，搭建过程参考：[ubuntu时间设置与ntp同步](/2014/10/28/ubuntu-ntp.html){:target="_blank"}。

## 角色分配

角色名称                |节点
----                    |--------
NameNode                |h196
Secondary NameNode      |h171
ResourceManager         |h171
slave                   |h196, h171, h189
job history server      |h196


只有3台机器，所以这里并没有让NameNode与ResourceManager做为专用节点，同样加入到从节点中了。线上环境的话还是应该NameNode与ResourceManager使用专用节点。

## jdk安装

选用的jdk版本为 7u71 的64位版，从官网下载 jdk-7u71-linux-x64.tar.gz 放到 VMBigData/resource 目录下，并做出如下命令操作

    cd ~/VMBigData/
    mkdir java
    tar zxf resource/jdk-7u71-linux-x64.tar.gz -C java/
    cd java/
    ln -s jdk1.7.0_71 default

这样子就安装好jdk了，并用default的软链接指向真正的jdk目录，使用default软链接可以方便以后更新jdk版本，只要把default软链接指向新的jdk目录即可，在后面的配置中当需要使用到 JAVA_HOME 变量时，其值为 /home/work/VMBigData/java/default 。


## hadoop安装文件布署

本次安装所选取的hadoop版本，为stable的2.5.1版，从hadoop官网下载 hadoop-2.5.1.tar.gz 到 VMBigData/resource 目录下。

    cd ~/VMBigData
    tar zxf resource/hadoop-2.5.1.tar.gz -C hadoop/
    cd hadoop/
    ln -s hadoop-2.5.1 default


# hadoop完全分布式集群布署

## 配置免密码ssh登录

只需要配置主节点到从节点的免密码登录。从节点到主节点，及从节点之间，是不需要配置的。运行NameNode及ResourceManager的节点即为主节点，剩余的都是从节点。

另外也要设置好各主节点之间及**主节点自己到自己（即localhost）**的免密码登录。

在每一个节点上生成ssh密钥对，若是原来已经生成过，就不必再次生成了：

    ## 注意，这里使用 dsa 方式，而不是ssh默认常用的 rsa 方式
    ## 不采用默认的rsa方式，避免以后由于误操作重新生成密钥对时覆盖掉原来的，这样免密码登录就失效了
    ssh-keygen -t dsa -P '' -f ~/.ssh/id_dsa

将所有**主节点**的`~/.ssh/id_dsa.pub`文件内容集合成一个文件内容，并把该文件内容追加到所有节点的`~/.ssh/authorized_keys`文件内容后面。

## 配置从节点

将需要做为从节点的机器hostname或IP地址放到 hadoop 目录（即 ~/VMBigData/hadoop/default)下的`etc/hadoop/slaves`文件内容中，一行一个从节点，内容如下：

    h171
    h189
    h196


## 建立存储数据的相应目录

hdfs是分布式文件系统，不过最终数据还是存储在一个个节点的磁盘目录上，所以需要在相应的节点上建立相应的存放数据的目录，用于后面的配置文件中进行配置。

### pid文件存放目录

默认pid文件是存放在`/tmp`目录下，这样有时不够安全，所以建一目录用于存放，下面的配置中会提到如何修改该路径。

    mkdir -p /home/work/VMBigData/hadoop/data/hdfs/pid

另外为了保护pid文件，将该目录权限做如下修改：

    chmod 700 /home/work/VMBigData/hadoop/data/hdfs/pid

### NameNode节点

NameNode节点需要存放NameNode数据，本例中在196上的建立以下一个目录用于存放数据，实际生产环境中最好用一个专门存放数据的磁盘下的目录。

    mkdir -p /home/work/VMBigData/hadoop/data/hdfs/namenode


### Secondary NameNode节点

Secondary NameNode节点需要存放NameNode的备份数据，本例中为h171节点，用以下命令建立如下目录：

    mkdir -p /home/work/VMBigData/hadoop/data/hdfs/namesecondary

### DataNode节点

#### datanode数据

从节点均做为DataNode节点，本例中在每一个从节点中建立以下两个目录用于存放datanode的数据。实际生产环境中，最好用不同磁盘下有一个目录。

    mkdir -p /home/work/VMBigData/hadoop/data/hdfs/datanode1
    mkdir -p /home/work/VMBigData/hadoop/data/hdfs/datanode2

#### local-dirs目录

local-dirs目录，用于存放作业运行时的存放中间结果的本地目录，一般设置多个，分摊磁盘IO负载。

    mkdir -p /home/work/VMBigData/hadoop/data/hdfs/localdir1
    mkdir -p /home/work/VMBigData/hadoop/data/hdfs/localdir2

#### log-dirs目录

log-dirs目录，用于存放日志，一般也设置多个。

    mkdir -p /home/work/VMBigData/hadoop/data/hdfs/logdir1
    mkdir -p /home/work/VMBigData/hadoop/data/hdfs/logdir2


## 配置参数

**注意：当下面的所有配置文件配置好之后，需要将所有配置文件分发到所有节点的相应路径下。下面所用配置参数的值的具体设定，是结合考虑了系统环境限制的，真实线上环境时请根据需要重新配置某些参数值。**

### 配置参数说明的官方链接

下面是各文件的具体配置参数说明的官方链接：

- [core-site.xml](http://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-common/core-default.xml){:target="_blank"}
- [hdfs-site.xml](http://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-hdfs/hdfs-default.xml){:target="_blank"}
- [mapred-site.xml](http://hadoop.apache.org/docs/current/hadoop-mapreduce-client/hadoop-mapreduce-client-core/mapred-default.xml){:target="_blank"}
- [yarn-site.xml](http://hadoop.apache.org/docs/current/hadoop-yarn/hadoop-yarn-common/yarn-default.xml){:target="_blank"}


### 环境变量的配置

在`etc/hadoop/hadoop-env.sh`文件的前面设置以下内容：

    export JAVA_HOME=/home/work/VMBigData/java/default
    export HADOOP_PREFIX=/home/work/VMBigData/hadoop/default

### pid存放位置的配置

在`etc/hadoop/hadoop-env.sh`文件中做出以下变量值的修改。`HADOOP_PID_DIR`注释掉原来的再新增一行，`YARN_PID_DIR`原本就没有，直接新增即可:

    #export HADOOP_PID_DIR=${HADOOP_PID_DIR}
    export HADOOP_PID_DIR=/home/work/VMBigData/hadoop/data/hdfs/pid
    export YARN_PID_DIR=/home/work/VMBigData/hadoop/data/hdfs/pid

在`etc/hadoop/mapred-env.sh`文件中做出如下修改：

    #export HADOOP_MAPRED_PID_DIR= # The pid files are stored. /tmp by default.
    export HADOOP_MAPRED_PID_DIR=/home/work/VMBigData/hadoop/data/hdfs/pid

### core-site.xml

    <configuration>
        <property>
            <name>fs.defaultFS</name>
            <value>hdfs://h196.vm.com:9000</value>
        </property>
    </configuration>

### hdfs-site.xml

这里主要完成datanode及namenode目录位置的配置，还有Secondary NameNode的配置，把Secondary NameNode指定在另外一个节点上，这样对于数据安全性更高一些。

    <configuration>
        <property>
            <name>dfs.replication</name>
            <!-- 单机版的一般设为1，若是集群，一般设为3 -->
            <value>2</value>
        </property>
        <property>
            <name>dfs.namenode.name.dir</name>
            <!-- 创建的namenode文件夹位置，如有多个用逗号隔开。配置多个的话，每一个目录下数据都是相同的，达到数据冗余备份的目的 -->
            <value>/home/work/VMBigData/hadoop/data/hdfs/namenode</value>
        </property>
        <property>
            <name>dfs.datanode.data.dir</name>
            <!-- 创建的datanode文件夹位置，多个用逗号隔开，实际不存在的目录会被忽略 -->
            <value>/home/work/VMBigData/hadoop/data/hdfs/datanode1,/home/work/VMBigData/hadoop/data/hdfs/datanode2</value>
        </property>
    
        <!-- 配置Secondary NameNode在另外一个节点上，该节点也将作为主节点之一 -->
        <property>  
            <name>dfs.http.address</name>  
            <value>h196.vm.com:50070</value>  
            <description>Secondary get fsimage and edits via dfs.http.address</description>  
        </property>  
        <property>  
            <name>dfs.secondary.http.address</name>  
            <value>h171.vm.com:50090</value>  
        </property> 
        <property>
            <name>dfs.namenode.checkpoint.dir</name>
            <value>/home/work/VMBigData/hadoop/data/hdfs/namesecondary</value>
        </property>
    
    </configuration>


### yarn-site.xml


    <configuration>
        <property>
            <name>yarn.resourcemanager.hostname</name>
            <value>h171.vm.com</value>
        </property>
    
        <property>
            <name>yarn.log-aggregation-enable</name>
            <!-- 打开日志聚合功能，这样才能从web界面查看日志 -->
            <value>true</value>
        </property>
        <property>
            <name>yarn.log-aggregation.retain-seconds</name>
            <!-- 聚合日志最长保留时间 -->
            <value>86400</value>
        </property>
    
    
        <property>
            <name>yarn.nodemanager.resource.memory-mb</name>
            <!-- NodeManager总的可用内存，这个要根据实际情况合理配置 -->
            <value>2048</value>
        </property>
        <property>
            <name>yarn.scheduler.minimum-allocation-mb</name>
            <!-- MapReduce作业时，每个task最少可申请内存 -->
            <value>512</value>
        </property>
        <property>
            <name>yarn.scheduler.maximum-allocation-mb</name>
            <!-- MapReduce作业时，每个task最多可申请内存 -->
            <value>2048</value>
        </property>
    
        <property>
            <name>yarn.nodemanager.vmem-pmem-ratio</name>
            <!-- 可申请使用的虚拟内存，相对于实际使用内存大小的倍数。实际生产环境中可设置的大一些，如4.2 -->
            <value>2.1</value>
        </property>
        <property>
            <name>yarn.nodemanager.vmem-check-enabled</name>
            <value>false</value>
        </property>
    
        <property>
            <name>yarn.nodemanager.local-dirs</name>
            <!-- 中间结果存放位置。注意，这个参数通常会配置多个目录，已分摊磁盘IO负载。 -->
            <value>/home/work/VMBigData/hadoop/data/hdfs/localdir1,/home/work/VMBigData/hadoop/data/hdfs/localdir2</value>
        </property>
        <property>
            <name>yarn.nodemanager.log-dirs</name>
            <!-- 日志存放位置。注意，这个参数通常会配置多个目录，已分摊磁盘IO负载。 -->
            <value>/home/work/VMBigData/hadoop/data/hdfs/logdir1,/home/work/VMBigData/hadoop/data/hdfs/logdir2</value>
        </property>
        <property>
            <name>yarn.nodemanager.aux-services</name>
            <value>mapreduce_shuffle</value>
        </property>
        <property>
            <name>yarn.nodemanager.aux-services.mapreduce.shuffle.class</name>
            <value>org.apache.hadoop.mapred.ShuffleHandler</value>
        </property>
    </configuration>


### mapred-site.xml


    <configuration>
        <property>
            <name>mapreduce.framework.name</name>
            <value>yarn</value>
        </property>
        <property>
            <name>yarn.app.mapreduce.am.resource.mb</name>
            <!-- 默认值为 1536,可根据需要调整，调小一些也是可接受的 -->
            <value>768</value>
        </property>
        <property>
            <name>mapreduce.map.memory.mb</name>
            <!-- 每个map task申请的内存，每一次都会实际申请这么多 -->
            <value>512</value>
        </property>
        <property>
            <name>mapreduce.map.java.opts</name>
            <!-- 每个map task中的child jvm启动时参数，需要比 mapreduce.map.memory.mb 设置的小一些 -->
            <!-- 注意：map任务里不一定跑java，可能跑非java（如streaming） -->
            <value>-Xmx384m</value>
        </property>
        <property>
            <name>mapreduce.reduce.memory.mb</name>
            <value>640</value>
        </property>
        <property>
            <name>mapreduce.reduce.java.opts</name>
            <value>-Xmx512m</value>
        </property>
        <property>
            <name>mapreduce.tasktracker.map.tasks.maximum</name>
            <value>2</value>
        </property>
        <property>
            <name>mapreduce.tasktracker.reduce.tasks.maximum</name>
            <value>2</value>
        </property>
        <property>
            <name>mapred.child.java.opts</name>
            <!-- 默认值为 -Xmx200m，生产环境可以设大一些 -->
            <value>-Xmx200m</value>
        </property>
        <property>
            <name>mapreduce.task.io.sort.mb</name>
            <!-- 任务内部排序缓冲区大小 -->
            <value>128</value>
        </property>
        <property>
            <name>mapreduce.task.io.sort.factor</name>
            <!-- map计算完全后的merge阶段，一次merge时最多可有多少个输入流 -->
            <value>100</value>
        </property>
        <property>
            <name>mapreduce.reduce.shuffle.parallelcopies</name>
            <!-- reuduce shuffle阶段并行传输数据的数量 -->
            <value>50</value>
        </property>
        <property>
            <name>mapreduce.jobhistory.address</name>
            <value>h196.vm.com:10020</value>
        </property>
        <property>
            <name>mapreduce.jobhistory.webapp.address</name>
            <value>h196.vm.com:19888</value>
        </property>
    </configuration>



## 启动hadoop

**以下所有命令都是在hadoop目录（即 VMBigData/hadoop/default目录）下运行的。**

### 格式化namenode

**该操作只需首次的时候执行，以后重启hadoop时无需再次执行。**

    bin/hdfs namenode -format

### 在NameNode节点上启动NameNode守护进程

    sbin/hadoop-daemon.sh --script hdfs start namenode

### 启动所有从节点的DataNode守护进程

    sbin/hadoop-daemon.sh --script hdfs start datanode

### 在ResourceManager节点上启动ResourceManager守护进程

    sbin/yarn-daemon.sh start resourcemanager

### 启动所有从节点的NodeManager守护进程

    sbin/yarn-daemon.sh start nodemanager

### 启动MapReduce JobHistory Server，在设定为运行该服务的节点上

    sbin/mr-jobhistory-daemon.sh start historyserver

## 通过浏览器查看服务情况

一旦hadoop服务正常启动之后，可通过下列地址在浏览器查看服务

守护进程                        |web地址                    |备注
------                          |------                     |-------
NameNode                        |http://nn_host:port/       |端口默认为50070，本实例为：http://192.168.1.196:50070
Secondary NameNode              |http://snn_host:port/      |端口默认为50090，本实例为：http://192.168.1.171:50090
ResourceManager                 |http://rm_host:port/       |端口默认为8088，本实例为：http://192.168.1.171:8088
MapReduce JobHistory Server     |http://jhs_host:port/      |商品默认为19888，本实例为：http://192.168.1.196:19888




## 停止hadoop

### 在NameNode节点上停止NameNode守护进程

    sbin/hadoop-daemon.sh --script hdfs stop namenode

### 停止所有从节点的DataNode守护进程

    sbin/hadoop-daemon.sh --script hdfs stop datanode

### 在ResourceManager节点上停止ResourceManager守护进程

    sbin/yarn-daemon.sh stop resourcemanager

### 停止所有从节点的NodeManager守护进程

    sbin/yarn-daemon.sh stop nodemanager

### 停止MapReduce JobHistory Server，在设定为运行该服务的节点上

    sbin/mr-jobhistory-daemon.sh stop historyserver



# 安装运行过程中碰到的问题记录

### 运行job时碰到错误信息：`running beyond virtual memory limits`

这是因为hadoop的YARN默认打开了一个配置特性`yarn.nodemanager.vmem-check-enabled`，该特性会检查job运行时所需的虚拟内存容易在运行节点上是否可以提供，若不能提供则报该错误并终止任务执行。通过查找相关资料，CDH也是默认关闭此特性的，所以推荐把该特性在`yarn-site.xml`配置中设成`false`吧，不然的话，要算好内存量之间的关系还是相当麻烦的，反正我是折腾了好久也没设定正确的内存容量值。

### 错误：This token is expired. current time is XXXXX found YYYYY

这种错误属于节点时间不一致导致的，把所有节点的时间同步一下，最好加上ntp服务就好了。

