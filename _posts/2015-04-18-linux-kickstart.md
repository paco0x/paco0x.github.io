---
layout:     post
title:      Linux Kickstart 自动安装
subtitle:   "Unattended installation with kickstart"
date:       2015-04-18 21:00:00
author:     Liao
catalog:    true
header-img: img/post-bg-2015.jpg
permalink:  /linux-kickstart/
tags:
    - Basic
---

## 从系统安装说起
在 RHEL，CentOS，Fedora 等系统中，安装系统使用的程序名叫 `anaconda`，它属于 `FedoraProject`，由 Python 开发，能够提供图形或者文本界面用于系统安装。

在安装系统之前，计算机的硬件上可能是没有操作系统的，因此为了能够运行安装程序，需要一个临时的操作系统，引导开机，启动安装程序，在使用光盘安装操作 linux 系统（这里特指 RHEL 系列的系统）时，一共有两个阶段，分别为引导和安装。

### Stage 1
使用光盘引导时，系统启动过程为 POST(加电自检) ---> BIOS 进行硬件检测并载入光盘的 MBR ---> 光盘的引导程序为 isolinux.bin，它根据 isolinux.cfg 生成一个菜单。当用户选择安装操作系统后，引导程序加载内核（vmlinuz）和 initrd.img 文件，initrd.img 会在内存中生成一个临时的操作系统，为安装过程提供一个安装环境。当系统切换至 initrd 文件系统后，initrd.img 中的 init 进程调用 /sbin/loader 程序，loader 探测安装介质，加载光盘 /img/in-post/stage2.img (在 RHEL6 中叫 install.img )，切换到 stage2，stage2.img 的文件系统类型是 suqashfs，安装系统的程序 anaconda 就包含其中。

### Stage 2
stage2.img 是 SquashFS 类型文件系统，其中包含了安装程序 anaconda 和它的配置文件。anaconda 提供了安装过程的配置界面，它可以提供文本、图形等安装管理方式，并支持 kickstart 等脚本提供自动安装的功能。在安装系统之后会自动生成 /root/anaconda-ks.cfg 的配置信息，其中记录了安装系统所选取选项自动生成的，方便以后自动安装。

下面是 CentOS 6 安装时 stage 1 的菜单界面：

![](/img/in-post/linux-kickstart/stage1.png)

这个菜单中的选项都是由光盘中的 `/isolinux/isolinux.cfg` 文件设定的。在菜单界面按 `Esc` 键，可以进入命令行模式，输入 `linux`，效果等同于菜单的第一个选项，即安装 Linux 操作系统。

在命令行接口下，还可以使用其他一些命令：

	网络相关配置：
	ip=IPADDR
	netmask=MASK
	gateway=GATEWAY
	dns=DNS_IP
	ifname=NAME:MAC_ADDR

	指明 kickstart 文件路径
	ks=cdrom:/path/to/ks.cfg
	ks=http://server/path/ks.cfg
	ks=ftp://server/path/ks.cfg
	ks=nfs:server:/path/ks.cfg

	dd：加载需要的驱动

这里的 `ks` 指令可以指定一个 kickstart 配置文件的路径，`anaconda` 可以根据 kickstart 文件的配置完成自动化安装操作系统。

## Kickstart
在常规系统安装中的需要手动选定系统安装的各种选项，kickstart 文件定义了这些系统安装需要选择的选项，`anaconda` 读取 kickstart 文件后，就可以根据文件的设置来进行系统安装，而不需要人为的选择安装配置了。

Kickstart 文件可以在安装时通过网络获取，它支持 http，ftp，nfs 等协议，还可以将 kickstart 文件存放在安装介质中（如光盘镜像），在安装时从光盘中读取。

### 创建一个 kickstart 文件
在常规的系统安装结束后，anaconda 会根据本次系统安装的设置，生成一个与本次安装设置相同的 kickstart 文件，这个文件位于 /root/anaconda-ks.cfg，可以对这个文件进行修改供以后使用。

RHEL 系系统还提供了一个图形化配置 kickstart 文件的工具 `system-config-kickstart`，能够在图形界面下选择安装选项并将结果保存为 kickstart 文件。

kickstart 文件有三部分组成：

### 命令段
命令段分为必备命令和可选命令。

**必选命令**

	keyboard us									# 键盘类型设定
	lang en_US									# 语言设定
	timezone [--utc] Asia/Shanghai				# 时区选择
	reboot | poweroff | halt						# 系统安装完成后的操作（重启或关机）
	selinux --disabled | --permissive				# 是否启用 selinux
	authconfig --useshadow --passalgo=sha512	# 系统的认证方式，这里选择密码认证，加密算法为 sha512
	rootpw --iscrypted ....						# 加密后的 root 密码
	bootloader --location=mbr --driveorder=sda	# bootloader 的安装位置，这里选择安装至 mbr 中

**可选命令**

	install | upgrade								# 安装/升级 操作系统
	url --url=....									# 指明通过远程主机的 FTP 或 HTTP 路径来安装系统
	firewall --disabled | --enabled				# 是否开启防火墙
	firstboot --disabled | --enabled				# 系统第一次启动后是否进行用户配置
	text | graphical								# 安装界面为 文本/图形
	clearpart --linux | --all 						# 安装前清除系统的哪些分区，--all 表示清除所有分区
	zerombr									# 使用 clearpart --all 时，需要加上这个选项，否则安装过程会被暂停，需要手动选择
	part											# 分区设定
	part swap --size=2048						# 对 swap 进行分区的示例
	part /boot --fstype ext4 --size=100000		# 对 /boot 进行分区的示例
	part pv.<id> --size=...						# 创建一个 PV
	volgroup vgname pvname					# 创建 VG
	logval /home --fstype ext4 --name=home --vgname=vgname --size=1024		# 创建一个逻辑卷的示例
	%include									# 可以将其他文件的内容包含进 kickstart 文件中来，包含的文件必须是安装系统过程中能够访问的

### 软件包选择段
这里定义安装系统需要安装的软件包，@开头的表示包组，也可以指定单个包名，如：

	%packages
	@Base
	@Core
	@base
	@basic-desktop
	@chinese-support
	@client-mgmt-tools
	@core
	@desktop-platform
	@fonts
	@general-desktop
	@graphical-admin-tools
	@legacy-x
	@network-file-system-client
	@perl-runtime
	@remote-desktop-clients
	@x11
	lftp
	tree
	%end

### 脚本段
脚本分配安装前脚本和安装后脚本

`%pre` 表示安装前脚本，此时的 Linux 系统环境为微缩版环境，脚本应尽可能简单

这里的脚本通常用于查询一些系统信息，然后根据这些信息动态的设定安装配置，例如使用下面的脚本

	%pre
	# 设置分区的配置，配置 swap 大小为和 内存大小一样
	#!/bin/sh
	act_mem=`cat /proc/meminfo | grep MemTotal | awk '{printf("%d",$2/1024)}'`
	echo "" > /tmp/partition.ks
	echo "clearpart --all --initlabel" >> /tmp/partition.ks
	echo "part /boot --fstype=ext3 --asprimary --size=200" >> /tmp/partition.ks
	echo "part swap --fstype=swap --size=${act_mem}" >> /tmp/partition.ks
	echo "part / --fstype=ext3 --grow --size=1" >> /tmp/partition.ks
	%end

这个脚本在安装系统之前执行，它查询了系统的内存大小，并根据内存大小生成分区指令，存放在 /tmp/partitoin.ks 文件中。

然后在 kickstart 文件中包含这个 partition.ks 文件，就可以动态的设置分区大小了：

	%include /tmp/partitions.ks

`%post` 表示安装后脚本，此时的 Linux 系统环境为已经安装完成的系统。

这里的脚本可以进行一些系统安装后配置，如公钥注入，仓库配置，第三方软件安装，系统服务配置文件修改等等。


## 制作一个引导光盘
前面说过，系统的安装分为引导和安装两布。在 CentOS 6 的安装光盘中，`/isolinux` 目录中存放的是用于引导的引导程序，它负责引导系统，加载一个内核，进入临时系统，启动 `anaconda` 安装程序。随后安装程序会对系统进行一些初始设置，安装用户程序软件包。

那么，可以制作这样一个引导光盘，它仅负责系统引导，而安装系统所需的安装树全部放在远程主机中，光盘中还可以内置一个 kickstart 文件，并修改引导菜单的选项使其安装时自动使用 kickstart 文件安装。

制作过程如下：

1. 准备工作目录，如 `/tmp/iso`
2. 将系统安装光盘中的 `/isolinux` 目录复制至 `/tmp/iso` 目录中
3. 将预先制作好的 kickstart 文件也放入 `/tmp/iso` 目录中
4. 编辑 `/tmp/iso/isolinux/isolinux.cfg` 文件，使其在安装时直接使用 kickstart 配置文件

	在文件中找到 label linux 菜单项，在 append 指令后附加 ks 设置，如：

    	label linux
    	  menu label ^Install or upgrade an existing system
    	  menu default
    	  kernel vmlinuz
    	  append initrd=initrd.img ks=cdrom:/ks.cfg
5. 创建 iso 镜像：

    	mkisofs -R -J -T -v --no-emul-boot --boot-load-size 4 --boot-info-table
    	-V "CentOS 6.6 X86_64 boot disk" -b isolinux/isolinux.bin -c isolinux/boot.cat -o /root/boot.iso cdrom/

然后就可以使用 boot.iso 这个镜像来安装操作系统了，当然安装时需要能够通过网络访问到存有安装树的远程主机。
