---
title: linux下virtualbox里虚拟机不能识别U盘等移动设备的解决办法
date: 2014-06-21 16:00:00 +0800
tags:
- virtualbox
---

解决步骤如下：

1.打开linxu终端，输入命令


	### 下面命令中的your_account_name，换成你自己的用户名
	sudo usermod -G vboxusers -a your_account_name


这条命令是将你目前使用的用户加入到virtualbox的用户组中，这样这个用户上的资源在virtualbox中也能使用，这就意味着在这个用户下能识别的u盘，在virtualbox也能识别了。

2.重启系统，然后使用刚才命令中的用户进行登陆（这样作的目的是让virtualbox用户组成员更新一下，以便加入刚才命令中的用户到其用户组中）

3.打开virtualbox的settings设置，将usb选项卡中的 Enable USB 2.0 (EHCI) Controller 勾选上，当然在此之前，记得先安装好Guest Addtionals。

4.做完以上步骤后，以后当需要在虚拟机中使用U盘等移动设备时，在 USB 选项卡中相应添加即可，或右键点击右下角的USB接口图标，选择相应设备。
