---
title: ansible教程
date: 2014-11-05 15:08:00 +0800
published: false
tags:
- ansible
---

* toc
{:toc}


# ansible介绍

ansible是个什么东西呢？官方的title是“Ansible is Simple IT Automation”——简单的自动化IT工具。这个工具的目标有这么几项：让我们自动化部署APP；自动化管理配置项；自动化的持续交付；自动化的（AWS）云服务管理。它是基于 paramiko 开发的。这个paramiko是什么呢？它是一个纯Python实现的ssh协议库。因此ansible就是不需要在远程主机上安装client/agents，因为它是基于ssh来和远程主机通讯的。

## 主控节点要求

当前ansible**不支持windows系统**，可以支持的系统有Red Hat, Debian, CentOS, OS X, BSDs等，不过要求机器上至少**安装有2.6（包括）以上版本的python**。

## 被控节点要求

被控节点至少需要安装有2.4（包括）以上版本的python。

如果python版本低于2.5（包括）的话，还需要安装`python-simplejson`模块。






注意点：

- ansible的`raw`模块的运行不需要`python-simplejson`，所以可以通过`raw`模块来让被控节点安装上`python-simplejson`。
- 如果被控节点开记了`SELinux`特性，那么这些被控节点在使用与`copy/file/template`相关功能时，还需要安装`libselinux-python`模块。可以通过ansible的`yum`模块来进行远程安装。
- python 3与python 2还是有一些差别的，ansible还没有切换到python 3上。因为有一些linux发行版默认安装的python版本是3而不是2，那么，在这些系统上，应该安装上python 2，并设置`ansible_python_interpreter`的指向python 2。如果需要在这些机器上安装python 2，可以通过`raw`模块来做到。



# 安装

## 下载

本文时，ansible的放出的版本为 1.7.2，所以这里下载该版本：

    wget https://codeload.github.com/ansible/ansible/tar.gz/v1.7.2 -O ansible-1.7.2.tar.gz

或者，采用github上最新的源代码版本，官方说一般两个月才发布一次版本，所以有些bug不能很快地修复并发布新版本出来，对于不想等待的人来说，就用github上的`devel`分支的最新源码好了：

    git clone https://github.com/ansible/ansible.git

本教程中所采用的就是github上clone出来的最新源码。

## 编译


如果还未安装`pip`，通过以下命令安装：

    sudo easy_install pip

ansible需要以下python模块：

    sudo pip install paramiko PyYAML Jinja2 httplib2

编译安装ansible:
    
    git clone https://github.com/ansible/ansible.git
    cd ansible
    source ./hacing/env-setup

## 升级与更新

通过git方式下载源码时，在升级ansible时，别忘了升级更新子模块`submodules`:

    git pull --rebase
    git submodule update --init --recursive

## 测试例子

当执行了 env-setup 脚本之后，默认的清单文件为`/etc/ansible/hosts`，不过可以通过参数选项进行指定。

下面运行一个简单的测试例子，达到的效果就是ssh登录上本机并执行ping命令。

    echo "127.0.0.1" > ~/ansible_hosts
    export ANSIBLE_HOSTS=~/ansible_hosts

然后通过以下命令运行起来：

    ansible all -m ping --ask-pass

在执行此测试例子时，发觉需要安装上`sudo apt-get install sshpass`，sshpass就是可以把密码通过参数告知，而ssh是不允许的。另外，也需要先`ssh 127.0.0.1`，把 127.0.0.1 加入到ssh的机器列表中。

# 讲解

## 关于远程连接

自从ansible 1.3版本之后，ansible会尽可能地通过原生的OpenSSH方式进行远程连接，这样将会开启ControlPersist、Kerberos特性，并使用`~/.ssh/config`里面的配置参数。不过，有些发行版的OpenSSH的版本过低，这样ansible就会使用由python实现的`paramiko`模块来替代OpenSSH。所以如果你想使用原生的OpenSSH的话，请升级过旧的OpenSSH版本，或让ansible运行在`accelerated`模式下。

对于低于 1.2版本的ansible，默认使用`paramiko`模块，如果将使用原生OpenSSH的话，请使用`-c ssh`参数进行指定。

有时，会碰到一些机器没有开启SFTP，那么可以通过配置文件指定使用SCP模式。

当与远程机器连接时，ansible会假设默认使用SSH keys方式，这是ansible推荐的一种方式，不过使用密码也是一样可行的。通过`--ask-pass`选项来开户密码，当需要使用`sudo`时，则使用`--ask-sudo-pass`选项。

通常来说，在内网当中选定一台做为主控节点是比较好的，而不是把主控节点放在其他的网段。

另外，ansible连接远程机器不一定非要通过SSH。传输是可定义的，有些选项是为了管理机器节点的本地事情的。有一种模式称为`ansible-pull`，可通过git方式下载playbook然后执行。


