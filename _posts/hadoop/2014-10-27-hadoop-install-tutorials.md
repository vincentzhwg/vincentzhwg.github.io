---
title: hadoop--安装介绍文档
date: 2014-10-27 10:40:00 +0800
tags:
- hadoop
- hdfs
- install
---

* toc
{:toc}

# 机器环境

- 3台处于同一网段的虚拟机
- 所有机器节点均能访问外网
- 系统均为 ubuntu12.04 LTS server x86-64
- 所有机器节点的登录帐户为 work ，但密码并不一定一致
- 所要安装的hadoop版本为stable版：2.5.1





# 前置准备

**前置准备中的操作，在每一个节点中都需要做。**当你有太多节点的时候，考虑使用一些方便的配置工具来完成。

## 目录

所有节点均使用work帐户，即$HOME路径为 /home/work ，以后所有的资源及文件均会在 $HOME 目录下的 VMBigData 目录下，即后面所说的 **VMBigData** 目录均指 /home/work/VMBigData 。

在 VMBigData 下建立 resource 目录，存放所有原始资源文件，如jdk安装文件，hadoop安装包等。

## 安装ssh, rsync

本机使用的是ubuntu，又可以访问外网，所以使用以下命令完成安装：

- sudo apt-get install ssh
- sudo apt-get install rsync

## DNS安装

三台机的ip地址分别如下：

- 192.168.1.171
- 192.168.1.189
- 192.168.1.196

为了方便，在 192.168.1.196 上安装了DNS服务，DNS安装请参考 [ubuntu搭建本地网DNS服务器](/2014/07/10/ubuntu-dns-bind.html){:target="_blank"} 。分配如下的域名:

- h171.vm.com : 192.168.1.171
- h189.vm.com : 192.168.1.189
- h196.vm.com : 192.168.1.196

将每一台机器的hostname均设置为自己的域名，如 192.168.1.171 的hostname即为 h171.vm.com 。

在后面的记录中，均使用节点简称来指某一个节点，如h171指h171.vm.com，h189指h189.vm.com，h196指h196.vm.com。

## 所有节点的时间同步

hadoop对于时间还是有一些要求，每一个节点的时间需要一致，所以最好搭建一个ntp服务，搭建过程参考：[ubuntu时间设置与ntp同步](/2014/10/28/ubuntu-ntp.html){:target="_blank"}。

## jdk安装

选用的jdk版本为 7u71 的64位版，从官网下载 jdk-7u71-linux-x64.tar.gz 放到 VMBigData/resource 目录下，并做出如下命令操作

    cd ~/VMBigData/
    mkdir java
    tar zxf resource/jdk-7u71-linux-x64.tar.gz -C java/
    cd java/
    ln -s jdk1.7.0_71 default

这样子就安装好jdk了，并用default的软链接指向真正的jdk目录，使用default软链接可以方便以后更新jdk版本，只要把default软链接指向新的jdk目录即可，在后面的配置中当需要使用到 JAVA_HOME 变量时，其值为 /home/work/VMBigData/java/default 。


# hadoop安装

本次安装所选取的hadoop版本，为stable的2.5.1版，从hadoop官网下载 hadoop-2.5.1.tar.gz 到 VMBigData/resource 目录下。

## 每个节点的准备工作

### 解压布署hadoop

    cd ~/VMBigData
    tar zxf resource/hadoop-2.5.1.tar.gz -C hadoop/
    cd hadoop/
    ln -s hadoop-2.5.1 default

### 配置 `JAVA_HOME` 和 `HADOOP_PREFIX` 变量

编辑 hadoop 目录（即 ~/VMBigData/hadoop/default 路径）下的 etc/hadoop/hadoop-env.sh 文件，修改或定义以下变量：

    # set to the root of your Java installation
    export JAVA_HOME=/home/work/VMBigData/java/default

    # Assuming your installation directory is /home/work/VMBigData/hadoop/default
    export HADOOP_PREFIX=HADOOP_PREFIX=/home/work/VMBigData/hadoop/default

配置正常的情况下，在 hadoop 目录下，运行以下命令，应该会看到输出的相应提示，如若没有，查找下是不是哪里安装或配置错了。
    
    bin/hadoop


### hadoop安装有三种模式

- Local (Standalone) Mode : 本地模式
- Pseudo-Distributed Mode : 伪分布式模式
- Fully-Distributed Mode : 完全分布式模式



## 本地模式

本地模式下，随便找一个节点即可，不需要其他节点参与，这里选取 h171 节点吧。

默认情况下，hadoop安装包被配置为非分布式模式，就只是一个运行的java进程，运行完成后即退出，这对于调试是有用的。

下面的例子，使用 hadoop 目录下 conf 目录里的文件做为输入，用给出的正则表达式进行匹配并输出匹配到的内容，输出内容被写到所给出的输出路径里。

    cd ~/VMBigData/hadoop/default

    ## input做为输入目录
    mkdir input
    cp etc/hadoop/*.xml input/

    ## 执行example的命令，输出目录不必事先创建，在运行中会创建
    bin/hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-2.5.1.jar grep input output 'dfs[a-z.]+'

这样子执行完成后，成功的话，会在hadoop目录下发现一个output目录，进去可以看到所得到的输出。


## 伪分布式模式

hadoop能够在单节点上以伪分布模式运行，其中每一个hadoop守护进程运行在不同的java进程中。这里选取h171节点。

### 配置

使用以下配置文件

`etc/hadoop/core-site.xml`

    <configuration>
        <property>
            <name>fs.defaultFS</name>
            <value>hdfs://localhost:9000</value>
        </property>
    </configuration>

---------

`etc/hadoop/hdfs-site.xml`

    <configuration>
        <property>
            <name>dfs.replication</name>
            <value>1</value>
        </property>
    </configuration>


### 设置localhost的免密码登录

先验证下对于localhost是否已经配置了免密码登录，运行以下命令，看是否可以不用输密码就可以ssh登录上
    
    ssh localhost

如果不能免密码登录，请照下面命令配置免密码登录

    ## 如果原来已经有 id_dsa 的话，可以不用再次生成，直接使用已有的
    ## 注意，这里使用 dsa 方式，而不是ssh默认常用的 rsa 方式
    ssh-keygen -t dsa -P '' -f ~/.ssh/id_dsa    
    cat ~/.ssh/id_dsa.pub >> ~/.ssh/authorized_keys

这样配置完成后，再次试验是否可以免密码登录localhost。

### 执行一个演示例子

1.在hadoop目录(即~/VMBigData/hadoop/default)下，执行

    bin/hdfs namenode -format

2.启动NameNode与DataNode的守护进程

    sbin/start-dfs.sh

hadoop守护进程的日志输出被输出到 $HADOOP_LOG_DIR 目录下，默认即为 $HADOOP_HOME/logs 下，若原先没有logs目录，会自动创建。

3.启动成功后，可通过浏览器来查看NameNode的情况，本例子中使用h171节点，所以访问地址为 `http://192.168.1.171:50070`

4.在HDFS上创建执行MapReduce任务所需的目录及数据

    bin/hdfs dfs -mkdir /user
    bin/hdfs dfs -mkdir /user/work      ## 这里user后面用work，是因为当前用的是work帐户

5. 拷贝任务的输入数据到hdfs上

    bin/hdfs dfs -put etc/hadoop input

6.同单例模式时一样，执行同一个演示例子，只是这次是在hdfs上进行输入及输出

    bin/hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-2.5.1.jar grep input output 'dfs[a-z.]+'

7.检查任务的输出

从hdfs上把输出复制到本地后再检查
    
    bin/hdfs dfs -get output output
    cat output/*

或者直接在hdfs上检查

    bin/hdfs dfs -cat output/*

8.当做完以上步骤后，就可以关闭守护进程了

    sbin/stop-dfs.sh

### 以YARN方式执行

上面的演示例子仍以传统的map reduce的方式执行的，下面说一下如何以YARN方式来执行。

下面执行的步骤假设你已经把上面演示例子的1~4的步骤都执行过了，然后继续做以下操作：

1.配置`etc/hadoop/mapred-site.xml`

可能一开始并没有这个配置文件，不过可以由官方提供的模板复制一个，执行以下命令

    cp etc/hadoop/mapred-site.xml.template etc/hadoop/mapred-site.xml

`etc/hadoop/mapred-site.xml`

    <configuration>
        <property>
            <name>mapreduce.framework.name</name>
            <value>yarn</value>
        </property>
    </configuration>

2.配置`etc/hadoop/yarn-site.xml`

    <configuration>
        <property>
            <name>yarn.nodemanager.aux-services</name>
            <value>mapreduce_shuffle</value>
        </property>
    </configuration>

3.启动ResourceManager与NodeManager守护进程

    sbin/start-yarn.sh

4.可通过浏览器查看ResourceManager信息，默认端口为8088。这里以h171为例，访问地址为`http://192.168.1.171:8088`

5.执行你想要执行的mapreduce程序或演示例子等

6.当操作完成后，通过以下命令关闭

    sbin/stop-yarn.sh

## 完全分布式模式

完全分布式模式，才是生产线上环境所使用的模式。

**通常，集群中有一个节点做为NameNode，还有另外一个节点做为ResourceManager,而且是专用的节点，它们都做为主节点。剩余的节点同时扮演DataNode与NodeManager,被称为从节点。**

### 以 Non-secure 模式运行

#### 配置文件

hadoop的配置主要通过两种重要类型的配置文件：

- 只读的默认配置：
    * core-default.xml
    * hdfs-default.xml
    * yarn-default.xml
    * mapred-default.xml
- 节点的指定配置：
    * etc/hadoop/core-site.xml
    * etc/hadoop/hdfs-site.xml
    * etc/hadoop/yarn-site.xml
    * mapred-site.xml

另外，可以通过 etc/hadoop/hadoop-env.sh 与 yarn-env.sh 来设置节点的指点配置，从而控制在 bin/ 目录下的可执行脚本。

#### 节点配置

为了配置hadoop集群，你需要配置hadoop守护进程运行时的环境变量。hadoop守护进程指 NameNode/DataNode 或 ResourceManager/NodeManager 。

#### 配置Hadoop守护进程的环境变量

hadoop管理员应该通过 `etc/hadoop/hadoop-env.sh` 与 `etc/hadoop/yarn-env.sh` 文件来做hadoop守护进程中环境变量的节点配置。至少，你应该指定 JAVA_HOME 路径，在每一个节点上。

大多数情况下，应该指定 `HADOOP_PID_DIR` 与 `HADOOP_SECURE_DN_PID_DIR` 的路径，这两个路径应该是运行hadoop守护进程的用户才拥有写权限，以免被非法用户攻击。

管理员对于不同的守护进程可以使用以下变量进行配置：

|守护进程                       |配置变量名称                       |
|---------                      |---------                          |
|NameNode                       |HADOOP_NAMENODE_OPTS               |
|DataNode                       |HADOOP_DATANODE_OPTS               |
|Secondary NameNode             |HADOOP_SECONDARYNAMENODE_OPTS      |
|ResourceManager                |YARN_RESOURCEMANAGER_OPTS          |
|NodeManager                    |YARN_NODEMANAGER_OPTS              |
|WebAppProxy                    |YARN_PROXYSERVER_OPTS              |
|Map Reduce Job History Server  |HADOOP_JOB_HISTORYSERVER_OPTS      |

例如，配置Namenode使用parallelGC,可在`hadoop-env.sh`中指定如下配置
    
    export HADOOP_NAMENODE_OPTS="-XX:+UseParallelGC ${HADOOP_NAMENODE_OPTS}"

另外一些有用的配置：
    
- HADOOP_LOG_DIR/YARN_LOG_DIR : 守护进程日志存放路径，如果路径原来不存在会自动创建。
- HADOOP_HEAPSIZE/YARN_HEAPSIZE : 最大可使用的堆容易，单位为 MB。若值设置为 1000，那么可用堆容量被设置为 1000MB，这也是默认值。如果你想为不同的守护进程设置不同的堆容量大小，可使用以下环境变量：

    |守护进程名称                   |配置变量名称                       |
    |------                         |---------                          |
    |ResourceManager                |YARN_RESOURCEMANAGER_HEAPSIZE      |
    |NodeManager                    |YARN_NODEMANAGER_HEAPSIZE          |
    |WebAppProxy                    |YARN_PROXYSERVER_HEAPSIZE          |
    |Map Reduce Job History Server  |HADOOP_JOB_HISTORYSERVER_HEAPSIZE  |

#### 在 Non-Secure 模式下配置hadoop的守护进程

本部分主要介绍在给定的配置文件中比较重要的配置选项。该部分内容请查阅hadoop官方文档：[Configuring the Hadoop Daemons in Non-Secure Mode](http://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-common/ClusterSetup.html#Configuring_the_Hadoop_Daemons_in_Non-Secure_Mode){:target="_blank"}。

下面是各文件的具体配置参数说明的官方链接：

- [core-site.xml](http://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-common/core-default.xml){:target="_blank"}
- [hdfs-site.xml](http://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-hdfs/hdfs-default.xml){:target="_blank"}
- [mapred-site.xml](http://hadoop.apache.org/docs/current/hadoop-mapreduce-client/hadoop-mapreduce-client-core/mapred-default.xml){:target="_blank"}
- [yarn-site.xml](http://hadoop.apache.org/docs/current/hadoop-yarn/hadoop-yarn-common/yarn-default.xml){:target="_blank"}


### hadoop机架感知（Rack Awareness）

HDFS与YARN组件具有机架感知能力。

NameNode与ResourceManager通过调用`resolve`接口（需要通过管理配置模块进行配置）来获取从节点的机架信息，接口解决域名或IP地址到机架id的映射关系。

使用哪个模块是通过配置项`topology.node.switch.mapping.impl`来指定的。模块的默认实现会调用`topology.script.file.name`配置项指定的一个的脚本/命令。 如果`topology.script.file.name`未被设置，对于所有传入的IP地址，模块会返回 /default-rack 作为机架id。

**机架感知功能默认是不起作用的，需要通过配置项并结合相应脚本才能使用该功能。**

### 监控NodeManager节点

hadoop给管理员提供了一种监控NodeManager节点的机制，即通过配置NodeManager定期运行指定的脚本，来判断NodeManager节点是否正常工作。

管理员判断一个节点是否处于正常运行状态，可以通过在指定脚本中执行任何的检查。**如果脚本检测到任何的不正常状态，它必须在标准输出中输出一行以 ERROR 开头的文本。**NodeManager节点定期执行该脚本，并检查脚本输出。如果脚本输出中包含上面所说的一行 ERROR 开头的文本，那么该节点被报告为`unhealthy`状态，将被ResourceManager列入其黑名单中，不会再有任务被分配到该节点。不过，该NodeManagera节点仍然继续定期运行检查脚本，如果后面恢复到`healthy`状态，该节点将从ResourceManager的黑名单中移除掉。这些状态的变化都可以从ResourceManager的web界面中查看得到。

以下参数可以用来配置监控NodeManager节点的脚本:

参数                                                    |值                                     
--------                                                |-------                               
yarn.nodemanager.health-checker.script.path             |节点监控脚本路径                     
yarn.nodemanager.health-checker.script.opts             |监控脚本参数                        
yarn.nodemanager.health-checker.script.interval-ms      |监控脚本运行间隔，毫秒为单位
yarn.nodemanager.health-checker.script.timeout-ms       |监控脚本运行的超时判定时间，毫秒为单位


当节点上的部分本地磁盘出错时，这个错误不应由健康监控脚本来给出，因为NodeManager本身具备定期检查本地磁盘是否正常的能力，特别是检查 nodemanager-local-dirs 与 nodemanager-log-dirs ，当坏掉的本地磁盘或目录个数超过`yarn.nodemanager.disk-health-checker.min-healthy-disks`所设定的阈值时，那么该节点将被标记为`unhealthy`并告知ResourceManager。**启动盘的正常与否，这个倒是应该由监控脚本来负责。**

### 从节点

通常，选取一个节点做为NameNode，另外一个节点做为ResourceManager，这两个节点都是专用的。剩余的节点同时扮演DataNode与NodeManager角色，这些剩余的节点被称为从节点，将从节点的hostname或IP，一行一个，列在`etc/hadoop/slaves`文件中。

### 日志

hadoop使用Apache log4j通过Apache Commons Logging framework来记录日志，编辑`etc/hadoop/log4j.properties`文件进行日志配置。

### 当所有必须的配置参数配置好之后，分发`HADOOP_CONF_DIR`目录下的所有内容到所有的节点上。

### 操作hadoop集群

#### 启动hadoop

为了启动hadoop集群，HDFS与YARN集群都需要启动。

格式化一个新的分布式文件系统：
    
    $HADOOP_PREFIX/bin/hdfs namenode -format <cluster_name>

用以下命令启动HDFS，需要在NameNode节点上执行该命令：
    
    $HADOOP_PREFIX/sbin/hadoop-daemon.sh --config $HADOOP_CONF_DIR --script hdfs start namenode

在所有的从节点上启动DataNode守护进程：

    $HADOOP_PREFIX/sbin/hadoop-daemon.sh --config $HADOOP_CONF_DIR --script hdfs start datanode

在ResourceManager节点上启动YARN:

    $HADOOP_YARN_HOME/sbin/yarn-daemon.sh --config $HADOOP_CONF_DIR start resourcemanager

在所有从节点上启动NodeManager守护进程：

    $HADOOP_YARN_HOME/sbin/yarn-daemon.sh --config $HADOOP_CONF_DIR start nodemanager

启动一个WebAppProxy服务。如果需要启动多个WebAppProxy服务（带负载均衡），那就应该在每一个需要的节点上运行启动命令：

    $HADOOP_YARN_HOME/sbin/yarn-daemon.sh start proxyserver --config $HADOOP_CONF_DIR

启动MapReduce JobHistory Server，在被设定运行该服务的节点上：

    $HADOOP_PREFIX/sbin/mr-jobhistory-daemon.sh start historyserver --config $HADOOP_CONF_DIR

#### 关停hadoop

在NameNode节点上运行以下命令停止NameNode:
    
    $HADOOP_PREFIX/sbin/hadoop-daemon.sh --config $HADOOP_CONF_DIR --script hdfs stop namenode

在所有从节点上停止DataNode守护进程：

    $HADOOP_PREFIX/sbin/hadoop-daemon.sh --config $HADOOP_CONF_DIR --script hdfs stop datanode

在ResourceManager节点上用以下命令停止ResourceManager:

    $HADOOP_YARN_HOME/sbin/yarn-daemon.sh --config $HADOOP_CONF_DIR stop resourcemanager

在所有从节点上停止NodeManager守护进程：

    $HADOOP_YARN_HOME/sbin/yarn-daemon.sh --config $HADOOP_CONF_DIR stop nodemanager

停止WebAppProxy。如果运行了多个WebAppProxy（带负载均衡）的话，那在每一个运行节点上：

    $HADOOP_YARN_HOME/sbin/yarn-daemon.sh stop proxyserver --config $HADOOP_CONF_DIR
    
停止MapReduce JobHistory Server，在运行该服务的节点上：

    $HADOOP_PREFIX/sbin/mr-jobhistory-daemon.sh stop historyserver --config $HADOOP_CONF_DIR


### 网页查看接口地址

当启动好hadoop集群之后，可通过以下网页地址进行查看服务是否正常。

守护进程                        |web地址                    |备注
------                          |------                     |-------
NameNode                        |http://nn_host:port/       |端口默认为50070
ResourceManager                 |http://rm_host:port/       |端口默认为8088
MapReduce JobHistory Server     |http://jhs_host:port/      |商品默认为19888

