---
layout:     post
title:      DNS 与 BIND
subtitle:   ""
date:       2015-04-26 23:00:00
author:     Liao
catalog:    true
header-img: img/post-bg-2015.jpg
permalink:  /dns-and-bind/
tags:
tags:
    - Basic
    - DNS
---

## DNS 简述
DNS 的全称是 Domain Name System，DNS 负责主机名字之间和互联网络地址之间的映射，在我们上网或者发送电子邮件的时候，一般都会使用主机名而不是 IP 地址，因为前者更容易记忆，但是对于计算机，使用数字（IP 地址）则更为方便。DNS 能够帮助我们将主机名转换成计算机更容易识别的 IP 地址。从而完成主机之前的通信。

### DNS 发展史

20 世纪 60年代，美国国防部开始资助试验性的广域计算机网络，称为 ARPAnet，它连接全美重要的研究机构。TCP/IP 协议是在 20 时机 80 年代初发展起来的，并迅速称为 ARPAnet 的标准主机网络协议。这个网络从只有屈指可数的几台主机发展到拥有成百上千的主机。

整个 20 世纪 70 年代， ARPAnet 只是一个拥有几百台主机的很小很友好的网络。仅仅需要一个名为 HOSTS.TXT 的文件就能容纳所有需要了解的主机信息：它包含所有连接到 ARPAnet 的主机的 名字－地址映射关系。与它极为相似的 Unix 主机中的 /etc/hosts，就是从 HOSTS.TXT 变化而来的。

HOSTS.TXT文件是由 SRI 的网络信息中心 （ Network Information Center， 简称NIC）负责维护，并且从一台主机 SRI-NIC 上分发到整个网络。 ARPAnet 的管理员们通常通过电子邮件的方式将他们的变更通知NIC， 同时还定期 FTP 到 SRI-NIC，以获取最新的 HOSTS.TXT 文件。每周进行一、两次更新，将这些变更汇编成新的 HOSTS.TXT 文件。但是随着 ARPAnet 的增长，这样的方法行不通了。HOSTS.TXT 文件大小的增长与 ARPAnet上主机数量的增加成正比。更重要的是，由于更新过程而引起的网络流量的增加更快： 每增加一台主机不仅仅意味着要在 HOSTS.TXT 文件中增加一行，更隐含着其他主机需要从 SRI-NIC 进行更新。由一台主机来管理HOSTS.TXT 文件就出现了的许多问题，DNS 就应运而生了。

### DNS 层级结构

DNS是一个层级的分布式的数据库，以 C/S 架构工作，它将互联网名称（域名）和 IP 地址的对应关系记录下来，可以为客户端提供名称解析的功能。它允许对整个数据库的各个部分进行本地控制，借助备份和缓存机制， DNS 将具有足够的强壮性。

DNS 数据库以层级的树状结构组织，最顶级的服务器被称为「根」（root），以 `.` 表示，它是所有子树的根。root 将自己划分为多个子域（subdomain），这些子域包括 `com`，`net`，`org`，`gov`，`net` 等等，这些子域被称为顶级域（Top Level Domain, TDL）。再进一步，各顶级域再将自己划分成多个子域，子域还可以在划分子域，最后树的叶子节点就是某个域的主机名。整个结构如下图所示：

![](/img/in-post/dns-and-bind/dns-hierarchy.png)

每个域的名称服务器仅负责本域内的主机的名称解析，如果需要解析子域的主机，就需要再向其子域的名称服务器查询。这样一来，无论主机在哪个域内，都可以从根开始一级一级的找到负责解析此主机名称的域，然后完成域名解析。

### DNS 解析步骤
下图是一个典型的 DNS 解析流程

![](/img/in-post/dns-and-bind/dns-query.png)

如图所示，客户端向某个 DNS 服务器发出请求，解析 www.kernel.org 的地址。

1. 服务器先向根名称服务器发出查询请求，根名称服务器的地址是内置在服务器端的，根名称服务器仅返回其下负责的 .org. 域的名称服务器的地址
2. DNS 服务器向 .org. 域的名称服务器发出查询请求，得到 kernel.org. 域的名称服务器的地址
3. DNS 服务器向 kernel.org. 域的名称服务器发出查询请求，该服务器发现请求的主机 www 就是本域下的主机，于是返回 www.kernel.org 主机的地址
4. DNS 服务器将得到的结果返回给客户端

在这次解析过程中，客户端仅向 DNS 服务器发送了一个请求，最后得到结果。而 DNS 服务器实际上发出了多次请求，从根开始依次查询最后得到结果。我们成前者为**递归查询**（recursion），后者被称为**迭代查询**（iteration）。从客户端到 DNS 的请求是通常是两段式的：先递归再迭代。

### 缓存
在网络通信中，域名的使用频率非常高，一个域名在某段时间内可能被反复的使用，如果每次使用都向 DNS 服务器查询，DNS 服务器也向其他服务器发出查询请求的话，将会消耗大量的网络带宽，并且速度也会非常慢。因此，一般在 DNS 客户端和服务器端都会有缓存，负责某个域的 DNS 服务器可以定义客户端缓存的时间。一个实际的查询往往是下面这样的

![](/img/in-post/dns-and-bind/dns-cache.png)

如图，当使用浏览器浏览网页时，需要用到 DNS 解析，而浏览器，操作系统都可能缓存已经解析过的结果，并且 ISP 提供商的 DNS 服务器也会缓存最近解析的结果，如果缓存中没有，再发出递归查询。通常，这样的 DNS 服务器被称为**缓存DNS服务器**


## DNS 客户端

### dig

dig 是 Linux 下常用的 DNS 查询工具，在 CentOS 系统中，由 `bind-utils` 软件包提供，它的使用方法为：

`dig -t RRT NAME [@NAME_SERVER]`

其中，RRT 表示资源记录类型，NAME 表示查询的地址，@NAME_SERVER 可以指定 DNS 服务器，如果不指定则使用操作系统默认的 DNS 服务器。

例如，查询 www.kernel.org 的 IP 地址：

	[root@bogon ~]# dig -t A www.kernel.org @172.16.0.1

	; <<>> DiG 9.8.2rc1-RedHat-9.8.2-0.30.rc1.el6 <<>> -t A www.kernel.org @172.16.0.1
	;; global options: +cmd
	;; Got answer:
	;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 35532
	;; flags: qr rd ra; QUERY: 1, ANSWER: 4, AUTHORITY: 3, ADDITIONAL: 4

	;; QUESTION SECTION:
	;www.kernel.org.			IN	A

	;; ANSWER SECTION:
	www.kernel.org.		600	IN	CNAME	pub.all.kernel.org.
	pub.all.kernel.org.	600	IN	A	199.204.44.194
	pub.all.kernel.org.	600	IN	A	198.145.20.140
	pub.all.kernel.org.	600	IN	A	149.20.4.69
	...(省略)

查询 google.com 域的名称服务器地址：

    [root@bogon ~]# dig -t NS google.com @172.16.0.1

    ; <<>> DiG 9.8.2rc1-RedHat-9.8.2-0.30.rc1.el6 <<>> -t NS google.com @172.16.0.1
    ;; global options: +cmd
    ;; Got answer:
    ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 9095
    ;; flags: qr rd ra; QUERY: 1, ANSWER: 4, AUTHORITY: 0, ADDITIONAL: 4

    ;; QUESTION SECTION:
    ;google.com.			IN	NS

    ;; ANSWER SECTION:
    google.com.		135181	IN	NS	ns4.google.com.
    google.com.		135181	IN	NS	ns2.google.com.
    google.com.		135181	IN	NS	ns3.google.com.
    google.com.		135181	IN	NS	ns1.google.com.

    ;; ADDITIONAL SECTION:
    ns4.google.com.		135181	IN	A	216.239.38.10
    ns1.google.com.		135181	IN	A	216.239.32.10
    ns2.google.com.		135181	IN	A	216.239.34.10
    ns3.google.com.		135181	IN	A	216.239.36.10

    ;; Query time: 137 msec
    ;; SERVER: 172.16.0.1#53(172.16.0.1)
    ;; WHEN: Sun Apr 26 18:04:16 2015
    ;; MSG SIZE  rcvd: 164

查询 IP 地址的反向解析记录：

    [root@bogon ~]# dig -x 8.8.8.8

    ; <<>> DiG 9.8.2rc1-RedHat-9.8.2-0.30.rc1.el6 <<>> -x 8.8.8.8
    ;; global options: +cmd
    ;; Got answer:
    ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 24762
    ;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 2, ADDITIONAL: 0

    ;; QUESTION SECTION:
    ;8.8.8.8.in-addr.arpa.		IN	PTR

    ;; ANSWER SECTION:
    8.8.8.8.in-addr.arpa.	51278	IN	PTR	google-public-dns-a.google.com.

    ;; AUTHORITY SECTION:
    8.in-addr.arpa.		51276	IN	NS	ns2.level3.net.
    8.in-addr.arpa.		51276	IN	NS	ns1.level3.net.

    ;; Query time: 2 msec
    ;; SERVER: 172.16.0.1#53(172.16.0.1)
    ;; WHEN: Sun Apr 26 18:05:18 2015
    ;; MSG SIZE  rcvd: 128

选择是否递归查询：

    dig +[no]recurse -t RRT NAME

从根开始追踪授权路径，显式每一个迭代查询的服务器，这对于调试很有帮助

    dig +[no]trace -t RRT NAME

### nslookup
nslookup 是一个交互式的 DNS 查询工具，它在 windows 和 linux 中均可以使用，在 CentOS 系统中由 `bind-utils` 软件包所提供。

输入 nslookup 即可进入交互式模式，输入域名或 IP 地址即可开始查询。

可以使用 `server SERVERNAME` 指定 DNS 服务器。

## BIND DNS 服务器搭建
BIND 是由 Berkely 大学研发的一款开源 DNS 服务器程序，同时也是最为流行的。在 CentOS 系统中，由 `bind` 软件包提供安装。

### bind 的配置文件
`/etc/named.conf`：定义 BIND 进程的工作属性和区域的定义

`/etc/rndc.key`：RNDC（Remote Name Domain Controller），远程控制的密钥文件

`/etc/rndc.conf`：远程控制的配置信息

`/var/named/`: 域数据文件

二进制程序：`/usr/sbin/named`

rndc 远程工具：`/usr/sbin/rndc`

语法和区域文件检查工具：`/usr/sbin/named-checkconf`，`/usr/sbin/named-checkzone`

**/etc/named.conf 的常用配置**

options 段，此处的配置在所有zone生效

	listen-on port 53 { any; };                                #监听的地址和端口
	directory    "/var/named";                             #区域文件的目录
	allow-query        { 10.10.0.0/8 };                    #仅允许查询的主机，默认是允许所有
	recursion          yes|no;                             #是否允许递归查询，默认是yes
	allow-recursion    { 10.10.0.0/8 };                    #允许递归查询的主机，默认是允许所有
	allow-query        { any; };                           #允许发出查询请求的主机
	notify             yes|no;                             #当主服务器serial number改变，通知从服务器同步
	forward            only|first;
			#定义转发规则, only表示服务器将转发至指定服务器，不论转后有没有结果, first表示服务器先转发至指定服务器，如果仍然没有结果，则开始从根服务器迭代查询
	forwarders         { 172.16.0.1; };                    #指定转发服务器，此时将转发所有非子域的请求至转发服务器
	querylog           yes|no;                             #是否开启查询日志

区域段

	zone "ZONE_NAME" IN {                                      #ZONE_NAME指区域的名称
		type {master|slave|hint|forward};                      #区域类型
		forward  {only|first};                                 #当区域类型为forward时的转发类型
		forwarders   { 172.16.0.1; };                          #转发服务器地址，此时仅转发转发区域的请求
		file "/PATH/TO/ZONE_FILE";                             #区域文件路径，相对于directory的相对路径
		masters { master1_ip;[ master2_ip]; };                 #从区域定义主服务器的地址
		allow-transfer     { none; }|{ 10.10.0.2; }           #是否允许传送区域，应该仅开启传送给从服务器，其他zone不要开启
	};

rndc

	key "rndc-key" {                                          #rndc的密钥名称
 		algorithm hmac-md5;                                  #加密算法
 		secret "8kp3nvAPk3YhySLwCBwDQA==";                   #密钥内容
	};

	controls {                                                #rndc控制服务
		inet 127.0.0.1 port 953                              #rndc监听的ip和端口
			allow { 127.0.0.1; } keys { "rndc-key"; };     #允许远程控制的主机及使用的密钥名称
	};

### 配置 example.com 域的正反向解析
首先，在 `/etrc/named.conf` 中开启监听端口，允许查询的主机，是否允许递归等选项。

    options {
    	listen-on port 53 { any; };
    	listen-on-v6 port 53 { ::1; };
    	directory 	"/var/named";
    	dump-file 	"/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
    	allow-query { any; };
    	recursion yes;

    	dnssec-enable no;
    	dnssec-validation no;
    };

在 `/etc/named.rfc1912.zones` 中添加正反向的区域定义

    zone "example.com" IN {
        type master;
        file "example.com.zone";
        allow-update { none; };
        allow-transfer { none; };
    };

    zone "0.10.10.in-addr.arpa" In {
        type master;
        file "10.10.0.zone";
        allow-update { none; };
        allow-transfer { none; };
    };

在 `/var/named/` 目录下创建定义的区域文件

`example.com.zone` 文件

	$TTL 1D
	@	IN	SOA	ns1		admin.example.com (
					2015042601 ; 版本序列号
					1D ; 主从架构时，从服务器的同步间隔
					2H ; 同步失败的重试间隔
					3D ; 同步失败超时时间，超时后从服务器变得不可用
					3H ) ; DNS 查询的否定回答（即查询不到结果）的缓存时间
			IN	NS	ns1
			IN	NS	ns2
			IN	MX 10	mail
	ns1		IN	A	10.10.0.1
	ns2		IN	A	10.10.0.2
	mail	IN	A	10.10.0.2
	www		IN	A	10.10.0.1
	news	IN	A	10.10.0.100
	games	IN	A	172.16.0.100

此文件中的条目被称作资源记录（Resource Record），其意义如下：

	$TTL	# BIND 内部的宏，表示缓存时间
	@		# BIND 内部的红，表示区域的域名
	IN		# 表示 INTERNET 类型
	SOA     # Start Of Authority，起始授权记录；一个区域解析库有且仅能有一个SOA记录，而必须为解析库的第一条记录；
	A       # 将主机名解析为 IP 地址
	PTR     # PoinTeR 反向解析，将 IP 地址解析为主机名
	NS      # Name Server，专用于标明当前区域的DNS服务器
	CNAME   # Canonical Name，别名记录
	MX      # Mail eXchanger，邮件交换器

`10.10.0.zone` 文件

    $TTL 1D
    @   IN  SOA ns1.example.com.  admin.example.com (
    							2015042601
    							1D
    							2H
    							3D
    							1D )
    	IN  	NS  	ns.example.com.
    1   IN  	PTR 	ns1.example.com.
    2   IN  	PTR 	ns2.example.com.
    2	IN		PTR		mail.example.com.
    1	IN		PTR		www.example.com.
    100	IN		PTR		news.example.com.

最后，两个文件的权限应该如下：

    [root@bogon named]# ll example.com.zone 10.10.0.zone
    -rw-r----- 1 root named 456 Apr 26 18:44 10.10.0.zone
    -rw-r----- 1 root named 213 Apr 26 18:33 example.com.zone

启动 named 服务即可工作。测试工作是否正常：

    [root@bogon named]# dig -t A www.example.com @10.10.0.1

    ; <<>> DiG 9.8.2rc1-RedHat-9.8.2-0.30.rc1.el6 <<>> -t A
    ...(省略)
    ;; QUESTION SECTION:
    ;www.example.com.		IN	A

    ;; ANSWER SECTION:
    www.example.com.	86400	IN	A	10.10.0.1

    ;; AUTHORITY SECTION:
    example.com.		86400	IN	NS	ns.example.com.

    ;; ADDITIONAL SECTION:
    ns.example.com.		86400	IN	A	10.10.0.1
    ...(省略)

### 配置主从复制
主从复制时，从 DNS 只需要从主 DNS 将区域文件传输到从服务器即可

1. 从服务器应该为一台独立的名称服务器；
2. 主服务器的区域解析库文件中必须有一条NS记录是指向从服务器；
3. 从服务器只需要定义区域，而无须提供解析库文件；解析库文件应该放置于/var/named/slaves/目录中;
4. 主服务器得允许从服务器作区域传送；
5. 主从服务器时间应该同步，可通过ntp进行；
6. bind程序的版本应该保持一致；否则，应该从高，主低；

这里，从服务器的 IP 为 10.0.0.2，将主服务器的 "example.com" 域的配置中的 `allow-transfer` 定义为：

    alllow-transfer { 10.10.0.2; };

在从服务器中的 `/etc/named.rfc1912.zones` 中定义从区域文件

	zone "example.com" IN {
		type slave;
		masters { 10.10.0.1; };
		file "slaves/example.com.zone";
		allow-transfer { none; };
	};

	zone "0.10.10-in-addr.arpa" IN {
		type slave;
		masters { 10.10.0.1; };
		file "slaves/10.10.0.zone";
		allow-transfer { none; };
	};

启动服务即可，如果主服务器的区域文件发生了更新，且序列号增加，主服务器会自动通知从服务器进行更新。

### 配置子域授权

在 example.com 域中，可以将 linux.example.com 域授权给其他名称服务器解析，这样，当请求 linux.example.com 域中的主机时，将由子域的名称服务器负责解析。授权的方式是在 example.com 域的区域文件中增加其子域的 NS 记录和对应的 A 记录。

在 `example.com` 区域文件中，增加这样两行

    linux   	IN  NS  ns.linux
    ns.lixnux   IN  A   10.10.0.3

之后就可以在 10.10.0.3 这台主机上，定义 linux.example.com 域并建立区域文件了。

### 转发 DNS
定义转发的意义在于，如果一个结果 DNS 服务器中没有定义相关的域，那么 DNS 可以将这个查询转发到其他的 DNS 服务器。被转发的服务器需要能够为请求者做递归查询，否则，转发请求不予进行。

转发选项可以定义在 options 段中，也可以定义在 zone 域的定义中。

全部转发：凡是对非本机所有负责解析的区域的请求，统统转发给指定的服务器

	Options {
		forward {first|only}
		fowwarders
	}

区域转发：仅转发对特定的区域的请求至某服务器

	zone "ZONE_NAME" IN {
		type forward;
		forward {first|only}
		forwarders
	}

### 视图
DNS的视图功能可以根据请求客户端的来源地址进行判断，不同的客户端 IP 地址对同一个地址的解析结果可以不同。这通常被称为智能 DNS。智能 DNS 的意义是，对来源 IP 地址按照运营商的不同，地域的不同，解析不同的结果，解析的结果通常是相对于客户端访问来说最为快速的 IP 地址（例如离客户端所在地最近）。智能 DNS 通常被用来构建 CDN 服务。

视图的定义方式是

	view VIEW_NAME {
		match-clients {    };
		zone ....
		....
	};

注意：

1. 一旦启用了view，所有的zone都只能定义在view中；
2. 仅有必要在匹配到允许递归请求的客户所在view中定义根区域；
3. 客户端请求到达时，是自上而下检查每个view所服务的客户端列表；

下图是一个使用 DNS view 功能对用户分类以提供更优服务器进行响应的例子：

![](/img/in-post/dns-and-bind/dns-view.png)
