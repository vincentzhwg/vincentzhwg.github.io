---
title: hbase备忘01--安装与运行
date: 2014-10-24 15:35:00
published: false
tags:
- hbase
- install
---


## 分布式模式安装

### 确保各节点之间的ssh无密码登录

在各机器节点上生成自己的ssh的key,使用 `ssh-keygen` 命令即可，若是原本已经有key了，即不必再次生成。然后，将key的公钥分发到其他节点上。**别忘了一点，节点自己登录自己的localhost，即`ssh-copy-id localhost`。**

### 配置master节点上的`conf/regionservers`

`conf/regionservers`配置文件，只需要配置master节点即可。




