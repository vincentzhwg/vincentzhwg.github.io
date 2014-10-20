---
title: linux下解决跳板机ssh登录与传输问题记要
date: 2014-05-21 16:00:00 +0800
tags:
- ssh
---

## ssh的control master特性的配置

公司开发环境限制，需要先登录跳板机，再ssh登录到真正需要操作的服务器上，而且跳板机的登录密码还需要一个随机token码，甚是麻烦呀。在windows下有SecureCRT，比较方便好用，虽说也有linux版本，但有30天的限制，也不好破解，毕竟都用了linux，那还是要尊重一下知识产权的。本文就是记录有没有什么好方法，在linux下也能方便地解决跳板机登录ssh的问题记要。

通过几番搜索，找到一个好办法，那就是利用ssh的controlmaster特性，结合 ~/。ssh/config 的配置来使用，但还未达到最终目的，可以直达真正想要到达的服务器，后续再找办法吧。




编辑 ~/.ssh/config 文件，示例内容如下

    Host a.b.com XXX
        HostName a.b.com
        User AAA
    
    Host *
        ServerAliveInterval 300
        ControlMaster auto
        ControlPath ~/.ssh/master-%r@%h:%p
        ControlPersist yes

- 第1行与第2行中的a.b.com替换成真正的域名或IP地址。
- 第1行的XXX改为你想要用的别名，毕竟方便记忆，还少敲几下键盘，还是好的。
- 第3行的AAA替换成真正登录时所使用的用户名。
- 第4行的host *表示它所设置的选项将适用到所有连接上。

通过controlmaster特性，只需要第一次登录时输入密码，后续再连接时无需密码直接就连接上了。

#### 几个有用的controlmaster命令

    ###第一次建立master连接时可采用以下命令，会在建立完master连接后回到当前本机的终端下
    ###当然，如果你就是想留在ssh机上，那就正常的命令连接就好了
    ssh -Nf XXX
      
    ### 检查master连接是否建立
    ssh XXX -O check
      
    ###结束master连接
    ssh XXX -O exit

---------------------------------------------

## 传输文件的解决办法：zssh

在自己的linux机上，如ubuntu等，安装上zssh，先用zssh登陆上跳板机，再在跳板机上ssh到相应服务器，然后ctrl+@，就可以相应上传下载文件了，先记着，后续再补详细资料。

#### 上传本地文件到服务器

    #在服务器上先cd至相应要放上传文件的目录之后
    
    rz -bye                 //在远程服务器的相应目录上运行此命令，表示做好接收文件的准备
    
    ctrl+@                  //运行上面命令后，会出现一些乱码字符，不要怕，按此组合键，进入zssh

    zssh >                  //这里切换到了本地机器

    zssh > pwd              //看一下本地机器的目录在那

    zssh > ls               //看一下有那些文件

    zssh > sz 123.txt       //上传本地机器的当前目录的123。txt到远程机器的当前目录


#### 下载服务器文件到本地

    sz filename             //在远程机器上，启动sz， 准备发送文件
                            //看到一堆乱码，不要怕，这会按下组合键
    ctrl+@

    zssh > pwd              //看看在那个目录，cd 切换到合适的目录
    
    zssh > rz -bye          //接住对应的文件
