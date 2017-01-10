---
layout:     post
title:      "RPM 入门"
subtitle:   "Redhat Package Management System"
date:       2014-11-13 23:00:00
author:     Liao
catalog:    true
header-img: img/post-bg-2015.jpg
permalink:  /linux-rpm/
tags:
    - Basic
---

这篇文章主要讲 RPM 软件包管理器的使用。

## 软件包的演变史

最早期时，软件包是一些可以运行的程序组成的集合，可能还要加上若干配置文件和动态库。例如，程序员将针对某个平台编译好的二进制文件、程序所依赖的动态库文件（如 `.so` 和 `.dll` 为扩展名的文件）以及配置文件复制到一个目录中，这个目录就可以称为一个软件包。

为了保证使用的软件包能够方便且快速地复制到别的机器上， 人们开始选用压缩文件的方式来封装软件包。 比如通过 `tar` 或者 `gzip` 压缩后得到 `.tar.gz`、 `.rar` 或者 `.zip` 格式的文件， 这时我们就获得了一个较为高级的软件包。

再往后发展， 就出现了更高级的软件包， 比如 `.rpm`、 `.bin` 或者 `.deb` 格式的软件包。 这些格式的软件包， 相对于压缩格式的软件包又有了更进一步的发展， 它们不仅支持文件压缩功能， 还有**依赖维护**、 **脚本的嵌入**等功能。RedHat 公司开发贡献的 RedHat Package Manager（RPM） 可以说是这些高级别软件包中最典型的一个。

## RPM 软件包的功能
RPM 软件包的功能如下：

- 存储和数据压缩
- 文件安装
- 配置文件生成
- 系统服务注册
- 软件依赖检查和依赖输出

### 存储数据压缩
RPM具有软件包的基本功能——数据压缩存储，RPM 安装列表中的文件在按照某个指定的算法（如 `gzip`）压缩后，作为最终 RPM 文件的一个数据块，与其他控制信息存储进同一个文件中。最终所有的数据都存储在同一个 RPM 文件中。

### 文件安装
文件安装是软件包的一个基本功能，它将压缩的文件解压至目标操作系统上。安装过程中，还可能动态生成一些文件，并安装到系统中。

### 配置文件生成
配置文件既可能是预先写好的静态文件，也可能是根据安装环境动态生成的文件。

### 系统服务注册
使用 rpm 安装一些软件包，比如 `apache`，`mysql-server` 等，在安装完成后，目录 `/etc/init.d/ `下会生成一个服务启动脚本文件，而且此服务还可能被加入到系统的自动启动服务中。

### 软件依赖检查
大多数程序都会依赖其他组件，比如数据库操作程序可能需要 `libmysql` 的支持。为了保证每个软件在安装后都能正常运行，在安装过程中，软件安装程序需要对该软件包所依赖的所有元素进行检查。

### 其他功能
RPM 还有一个重要功能就是对嵌入脚本的支持：它支持在安装软件或者卸载软件的过程中，执行用户预定义的指令。常用的脚本执行点如下：

- pre install
- post install
- pre uninstall
- post uninstall

pre/post install 表示在安装之前或之后；pre/post uninstall 表示在卸载之前或者之后。

除此之外，RPM 包还可以支持对源包进行数字签名，在安装时可以使用公钥验证 RPM 包的合法性等等。

### RPM 包的命名方式
以 `httpd-2.2.15-39.el6.centos.x86_64.rpm` 为例，这里 `httpd` 表示软件名，`2.2.15` 表示主版本号，次版本号，发行版本号分别是`2`，`2`，`15`，`39.el6.centos` 表示 RPM 包的修订号和 OS 信息，`x86_64` 表示此软件包适用的平台，常见的有`i386`，`i586`，`x86_64` 等等。

## RPM 包管理命令的使用
### 安装
`rpm {-i|--install} [install-options] PACKAGE_FILE1..`

安装时可以使用 `-h` 显式安装进度，使用 `-v` 显示详细信息。

	[root@localhost ~]# rpm -ivh httpd-2.2.15-39.el6.centos.x86_64.rpm
	Preparing...                ########################################### [100%]
	   1:httpd                  ########################################### [100%]

使用 `--test` 可以用于测试安装是否能够成功，而不实际安装。

在安装过程中，可能遇到软件包的依赖问题，而需要先安装其他软件包，这时可以使用 `--nodeps` 忽略依赖强制安装，但是这样安装的软件包通常也会因为依赖缺失而无法正常工作。

如果需要重新安装并覆盖原有的文件，可以使用 `--replacepkgs` 选项。

使用 `--force` 可以进行强制覆盖安装，它等同于`--replacepkgs, --replacefiles, 和 --oldpackage`。

### 升级
**升级或安装**

如果不知道一个软件包是否已经安装，并希望如果已经安装那么升级次软件包，使用 `-U` 选项。

    rpm {-U|--upgrade} [install-options] PACKAGE_FILE ...

如果仅仅希望升级软件包，使用 `-F` 选项

    rpm {-F|--freshen} [install-options] PACKAGE_FILE ...

升级软件包和安装软件包一样，可以使用 `--test`，`--nodeps`，`--force` 等选项。

示例：安装并升级 zsh 软件包

	[root@localhost rpm]# rpm -ivh zsh-4.3.10-7.el6.x86_64.rpm
	Preparing...                ########################################### [100%]
	   1:zsh                    ########################################### [100%]
	[root@localhost rpm]# rpm -Uvh zsh-4.3.10-9.el6.x86_64.rpm
	Preparing...                ########################################### [100%]
	   1:zsh                    ########################################### [100%]

如果想要将软件包降级到旧版本，使用 `--oldpackage` 选项

	[root@localhost rpm]# rpm -Uvh --oldpackage zsh-4.3.10-7.el6.x86_64.rpm
	Preparing...                ########################################### [100%]
	   1:zsh                    ########################################### [100%]

在升级软件包时，原来软件包的配置文件可能已经被修改，升级时，新版本的文件不会将老版本的配置文件覆盖，而是将新版本的配置文件加上 `.rpmnew` 后缀后保存。

**注意**：内核也是软件包，但是不要直接对内核进行升级（如果新的内核有兼容问题启动不了而旧内核又被覆盖就悲剧了），因为 Linux 允许多内核共存，所以可以直接安装多个不同版本内核。

### 卸载
`rpm {-e|--erase} [--allmatches] [--nodeps] [--test] PACKAGE_NAME ...`

通常使用 `rpm -e PACKAGE_ANEM` 即可简单卸载一个软件包。

使用 `--nodeps` 忽略依赖关系。`--test` 测试卸载。`--allmatches` 表示如果一个程序包同时安装多个版本，则次选项一次全部卸载之。

如果卸载正常，不会输出任何信息。

**注意**：如果程序包的配置文件安装后曾被修改，卸载时，此文件通常不会被删除，而是被重命名为 `.rpmsave` 后缀后留存。

### 查询：
查询使用 `-q` 选项，可以检查安装的所有包，还可以查看某包的详细信息。

`rpm {-q|--query} [select-options] [query-options]`

**查询某包是否已经安装**

`rpm -q PACKAGE_NAME...`

如：

	[root@localhost rpm]# rpm -q zsh
	zsh-4.3.10-9.el6.x86_64

**查询安装的所有包**

`rpm -qa`

**查询未安装包的信息**

在 `-q` 同时使用 `-p` 选项

注意：查询未安装包的信息指定的是 RPM 包的文件名而不是某个包的软件名。

### 查询选项

**查询某包的简要说明信息**

`rpm -qi PACKAGE_NAME`

如：

	[root@localhost rpm]# rpm -qi zsh
	Name        : zsh                          Relocations: (not relocatable)
	Version     : 4.3.10                            Vendor: CentOS
	Release     : 9.el6                         Build Date: Wed 05 Nov 2014 07:20:52 PM CST
	Install Date: Sat 11 Apr 2015 11:37:12 PM CST      Build Host: c6b8.bsys.dev.centos.org
	Group       : System Environment/Shells     Source RPM: zsh-4.3.10-9.el6.src.rpm
	Size        : 5009102                          License: BSD
	Signature   : RSA/SHA1, Wed 05 Nov 2014 08:05:42 PM CST, Key ID 0946fca2c105b9de
	Packager    : CentOS BuildSystem <http://bugs.centos.org>
	URL         : http://zsh.sunsite.dk/
	Summary     : A powerful interactive shell
	Description :
	The zsh shell is a command interpreter usable as an interactive login
	shell and as a shell script command processor.  Zsh resembles the ksh
	shell (the Korn shell), but includes many enhancements.  Zsh supports
	command line editing, built-in spelling correction, programmable
	command completion, shell functions (with autoloading), a history
	mechanism, and more.

这里显式了 `zsh` 这个包的各类元信息，如名字，版本，发行商，打包作者，描述信息等。

**查询软件包安装的文件列表**

`rpm -ql PACKAGE_NAME`

如：

	[root@localhost rpm]# rpm -ql zsh
	/bin/zsh
	/etc/skel/.zshrc
	/etc/zlogin
	/etc/zlogout
	/etc/zprofile
	/etc/zshenv
	/etc/zshrc
	/usr/lib64/zsh
	/usr/lib64/zsh/4.3.10
	...
	/usr/share/zsh/4.3.10/functions/zstyle+
	/usr/share/zsh/4.3.10/scripts
	/usr/share/zsh/4.3.10/scripts/newuser
	/usr/share/zsh/site-functions

使用 `rpm -qc PACKAGE_NAME` 可以查看软件包安装后生成的所有配置文件。

使用 `rpm -qd PACKAGE_NAME` 可以查看软件包安装后生成的所有说明文件和帮助文件。

**查看软件包制作时随版本变化的 changelog 信息**

`rpm -q --changelog PACKAGE_NAME`

**查看软件包提供的 capabilities （即输出给其他软件包的依赖）**

`rpm -q --provides PACKAGE_NAME`

**查看软件包所需的依赖**

`rpm -q --requires PACKAGE_NAME`

**查看软件包安装或卸载时执行的脚本**

`rpm -q --scripts PACKAGE_NAME`

如：

	[root@localhost rpm]# rpm -q --scripts zsh
	postinstall scriptlet (using /bin/sh):
	if [ ! -f /etc/shells ] ; then
	    echo "/bin/zsh" > /etc/shells
	else
	    grep -q "^/bin/zsh$" /etc/shells || echo "/bin/zsh" >> /etc/shells
	fi

	if [ -f /usr/share/info/zsh.info.gz ]; then
	# This is needed so that --excludedocs works.
	/sbin/install-info /usr/share/info/zsh.info.gz /usr/share/info/dir \
	  --entry="* zsh: (zsh).			An enhanced bourne shell."
	fi

	:
	preuninstall scriptlet (using /bin/sh):
	if [ "$1" = 0 ] ; then
	    if [ -f /usr/share/info/zsh.info.gz ]; then
	    # This is needed so that --excludedocs works.
	    /sbin/install-info --delete /usr/share/info/zsh.info.gz /usr/share/	info/dir \
	      --entry="* zsh: (zsh).			An enhanced bourne shell."
	    fi
	fi
	:
	postuninstall scriptlet (using /bin/sh):
	if [ "$1" = 0 ] ; then
	    if [ -f /etc/shells ] ; then
	        TmpFile=`/bin/mktemp /tmp/.zshrpmXXXXXX`
	        grep -v '^/bin/zsh$' /etc/shells > $TmpFile
	        cp -f $TmpFile /etc/shells
	        rm -f $TmpFile
	    fi
	fi

这里包含了安装后，卸载前/后脚本。

### 检验
还可以查询软件包安装之后的文件是否发生了改变

`rpm {-V|--verify} [select-options] [verify-options]`

如：

	[root@localhost rpm]# rpm -V httpd
	S.5....T.  c /etc/httpd/conf/httpd.conf

检验时使用了多个位表示文件的多个属性是否发生了变化：

	S 文件大小
	M 文件权限
	5 文件摘要信息（通常是 MD5 码）
	D 设备文件的主/次设备号
	L 软链接变化
	U 属主
	G 属组
	T 文件的 mtime
	P caPabilities

**程序包的合法性验证**

在软件包制作时，为了防止软件包被人修改植入后门，制作者可以使用自己私钥对软件包进行数字签名，安装者就可以使用公钥验证软件包的合法性。同时还可以使用摘要算法提取软件包的摘要信息用于验证软件包的完整性。

通常，RHEL 系的安装光盘中包含有用于验证其软件包合法性的公钥文件。

**导入公钥**

`rpm --import /path/to/RPM-GPG-KEY-FILE`

**验证合法性**

`rpm {-K|--checksig} PACKAGE_FILE`

### RPM 管理器的数据库
每次安装 rpm 包时，rpm 系统会将一些元信息存储在它的数据库中，使用 `rpm -q` 命令查询软件包的相关信息时将会查询这些数据库，数据库文件位于 `/var/lib/rpm` 目录中。如果 RPM 的数据库损坏，将会导致一些 RPM 数据丢失，一些功能将无法正常使用。

	[root@bogon ~]# file /var/lib/rpm/*
	/var/lib/rpm/Basenames:      Berkeley DB (Hash, version 9, native byte-order)
	/var/lib/rpm/Conflictname:   Berkeley DB (Hash, version 9, native byte-order)
	/var/lib/rpm/__db.001:       Applesoft BASIC program data
	/var/lib/rpm/__db.002:       386 pure executable
	/var/lib/rpm/__db.003:       386 pure executable not stripped
	/var/lib/rpm/__db.004:       386 pure executable
	/var/lib/rpm/Dirnames:       Berkeley DB (Btree, version 9, native byte-order)
	/var/lib/rpm/Filedigests:    Berkeley DB (Hash, version 9, native byte-order)
	/var/lib/rpm/Group:          Berkeley DB (Hash, version 9, native byte-order)
	/var/lib/rpm/Installtid:     Berkeley DB (Btree, version 9, native byte-order)
	/var/lib/rpm/Name:           Berkeley DB (Hash, version 9, native byte-order)
	/var/lib/rpm/Obsoletename:   Berkeley DB (Hash, version 9, native byte-order)
	/var/lib/rpm/Packages:       Berkeley DB (Hash, version 9, native byte-order)
	/var/lib/rpm/Providename:    Berkeley DB (Hash, version 9, native byte-order)
	/var/lib/rpm/Provideversion: Berkeley DB (Btree, version 9, native byte-order)
	/var/lib/rpm/Pubkeys:        Berkeley DB (Hash, version 9, native byte-order)
	/var/lib/rpm/Requirename:    Berkeley DB (Hash, version 9, native byte-order)
	/var/lib/rpm/Requireversion: Berkeley DB (Btree, version 9, native byte-order)
	/var/lib/rpm/Sha1header:     Berkeley DB (Hash, version 9, native byte-order)
	/var/lib/rpm/Sigmd5:         Berkeley DB (Hash, version 9, native byte-order)
	/var/lib/rpm/Triggername:    Berkeley DB (Hash, version 9, native byte-order)

可以看到这里有很多 Berkeley DB 格式的数据库文件和几个 __db 数据文件。

**重建数据库**

如果 RPM 的数据库损坏，首先可以尝试重建它，如果无法重建，那么需要重新初始化数据库。

`rpm --rebuilddb`  表示重建数据库

这个命令会从已安装的软件包提取信息重建数据库，它从 `/var/lib/rpm/Packages` 这个文件中提取信息，其他所有的数据库文件都可以由这个文件重建。如果 RPM 的数据库是完好的，这个命令不会重建，而是对数据库中未使用的条目进行空间回收。

`rpm --initdb`  创建一个新的 RPM 数据

如果已经没有其他别的办法了，`--initdb` 会创建一个新的空的 RPM 数据库。由于新建的数据库是空的，不要万不得已不要使用这个命令。










