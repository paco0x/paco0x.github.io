---
layout:     post
title:      "Cacti 使用中无数据的问题的总结"
subtitle:   "Debug in Cacti"
date:       2014-12-18 22:15:00
author:     Liao
catalog:    true
header-img: img/post-bg-2015.jpg
permalink:  /cacti-no-data-problem/
tags:
    - MySQL
    - Performance Tuning
---

Cacti 是 PHP 开发的网页程序，它能够周期性的执行能够取得数据的命令，并将取回的数据保存至 RRD 文件中，并利用 RRDtool 绘图并展示至网页中。

Cacti 的着眼点就是，收集某一时间的具体数据，并根据数据绘图，展示一个具体的走势，因此它的重点是绘图展示，而不是监控报警。

**RRDtool**

Cacti 底层使用 RRDtool 工具来进行数据的保存和图形的绘制，RRDtool 这个工具非常强大，但是使用方法复杂，Cacti 将 RRDtool 的使用抽象为网页程序，并提供一系列模版，使监控绘图变得简单。

详细了解 RRDtool 可以参考[这篇文章](http://bbs.chinaunix.net/forum.php?mod=viewthread&tid=864861&page=1&authorid=20054105)和[官方文档](http://oss.oetiker.ch/rrdtool/index.en.html)，了解了 RRDtool 的使用后 Cacti 的使用就变得易如反掌了。

在使用时，经常会出现一些无数据的情况，此时错误排查就需要了解底层的 RRDtool 的原理，这里总结一些常见的无数据问题。

## Cacti 获取数据的方式

Cacti 获取数据的方式有两种，监控端的脚本（可以是php, shell, perl 或其他脚本）或者 snmp 协议获取。

Cacti 会在固定的时间间隔启动轮询进程，使用定义好的方式获取被监控的数据，这种监控方式通常叫做**主动监控**。

执行轮询获取数据的进程叫做 `cmd.php` 它由 `poller.php` 调用，`poller.php` 在监控机上由计划任务定时执行。数据量多的时候，可以设置启动多个 `cmd.php` 进程，或者使用 `spine` 这个程序，它由 c 语言编写，效率更高，可以实现多进程加多线程。

- **主动监控** 的优点是被监控端通常不需要额外安装其他软件，一切数据由监控端主动来获取。

- **主动监控** 的缺点很明显，如果某些被监控端出现异常，获取不到数据或者数据获取延迟。那么监控端的轮询进程需要等待这些被监控数据获取超时或延迟，这样必然会阻塞轮询进程去获取其他数据，在轮询时间短或主机较多的情况下，可能出现在轮询周期已经结束，下一次轮询已经开始，而本次轮询还没有结束，造成某些没来得及获取。

在大规模的监控体系中，一般使用**被动监控**，客户端主动向监控机发送数据。**被动监控**一般需要在客户端安装监控代理程序，Nagios 和 Zabbix 可以实现被动监控，而 Cacti 只支持主动监控。

## 数据源无数据的常见原因

**客户端的问题**

客户端的 snmpd 服务未启动，snmp community 不正确，snmp 访问控制权限不正确，或者防火墙设置不正确都可能会影响 snmp 数据的获取。这种情况通常可以在监控机上使用 snmpwalk 等客户端来排查。

**监控端的问题**

1. 监控端的数据收集方法不正确，在 Cacti 中数据源页面，查看 Data Template，去对应的 Data Template 页面查看数据收集方法，使用 snmpwalk 或其他方式手动验证是否能够收集数据。

2. rrd 文件权限不正确，在 Data Sources 中查看对应 rrd 文件的路径，查看 rrd 文件的权限，Cacti 的轮询进程必须能够向此文件写入数据。

3. rrd 文件的属性设置不正确，rrd 文件可以对数据源的存放格式，聚合函数，最大值，最小值等进行设置，这些设置不当都可能造成数据源中无数据。

## rrd 文件属性设置不当造成无数据

这里我重点谈一谈 rrd 文件属性设置，因为这个而导致的无数据问题最不容易被发现，且网上的介绍也很少。

进入某个数据源页面，点击右上角的 `Turn On Data Source Debug Mode`

    /usr/bin/rrdtool create \
    /home/www/html/cacti/rra/23/477.rrd \
    --step 300  \
    DS:cpu_user:COUNTER:600:0:500 \
    RRA:AVERAGE:0.5:1:600 \
    RRA:AVERAGE:0.5:6:700 \
    RRA:AVERAGE:0.5:24:775 \
    RRA:AVERAGE:0.5:288:797 \
    RRA:MAX:0.5:1:600 \
    RRA:MAX:0.5:6:700 \
    RRA:MAX:0.5:24:775 \
    RRA:MAX:0.5:288:797 \

上面的数据是某个 Data Source RRD 文件的 debug 信息，其中 `DS:cpu_user:COUNTER:600:0:500` 一行为调试信息

各个字段的意义为：

    DS： 数据源
    cpu_user：数据源名称
    COUNTER：数据源类型，COUNTER 表示保存获取数据较上一次数据的相对值除以时间间隔，即每秒数据变化率
    600：心跳时间，即超过 600s 获取的数据为超时数据
    0：数据源的最小值
    500：数据源的最大值

在 Data Template 中查询此数据源获取数据的方法为使用 snmp 获取 `.1.3.6.1.4.1.2021.11.50.0` 这个 OID 的数据，这个数据的说明如下

    The number of 'ticks' (typically 1/100s) spent
    processing user-level code.
    
    On a multi-processor system, the 'ssCpuRaw*'
    counters are cumulative over all CPUs, so their
    sum will typically be N*100 (for N processors)

即此数据为用户空间 CPU 使用的嘀嗒数 (1/100s) * N，N 为 CPU 核数，此值为累加值，因此数据源使用 COUNTER 类型，计算出每秒的 CPU 使用率，但是一个问题是，当 CPU 核数超过 5 个时，此时这个值将大于 500，造成数据超出最大值，表现为 数据源中无数据。

这个问题的修复方法是修改数据源的最大值属性，在 Cacti 中无法直接修改已创建数据源的属性，需要使用 rrdtool 这个工具修改

修改的方法：

    rrdtool tune test.rrd --maximum cpu_user:U

表示将 test.rrd 这个数据库中的 cpu_user 这个数据源的最大值属性改为 U（无最大值限制）

查看属性的方法：

    rrdtool info test.rrd

可以看到修改后的最大值。

为了防止下次创建数据源时再次使用不正确的最大值属性，还应该将对应数据模版中的最大值限定设置为一个合理值。

目前我发现有这个问题的模版有：

    ucd/net - CPU Usage - Nice
    ucd/net - CPU Usage - System
    ucd/net - CPU Usage - User
    ucd/net - Memory - Buffers
    ucd/net - Memory - Cache
    ucd/net - Memory - Free

这些图的数据源修改设置后已经可以正确获取数据了：

![](/img/in-post/cacti-problem/cacti_pic1.png)

![](/img/in-post/cacti-problem/cacti_pic2.png)

![](/img/in-post/cacti-problem/cacti_pic3.png)

## Cacti 老版本 percona-plugin 的数据收集问题

percona-plugin 是使用 php script 来对被监控机进行数据收集，此 php 脚本的路径为 `<path_cacti>/scripts/ss_get_mysql_stats.php`，其中 <path_cacti> 表示 cacti的路径

由于监控系统中 percona-plugin 插件的版本较老，而 MySQL 5.5 中由于 SHOW ENGINE INNODB STATUS 的输出结构改变了，一些 InnoDB 相关的数据无法使用此脚本查询，导致一些数据源中无数据，比如 `X InnoDB Insert Buffer DT` 这个数据模版。

下载了新版本的 percona-plugin 后发现，这个脚本中的内部变量已经完全改了，接受的执行参数也不同，直接升级插件可能导致老数据源的数据无法获取。

一个解决办法是 将新版本中针对 MySQL 5.5 获取数据的新方法更新至老版本的脚本中，如下图：

![](/img/in-post/cacti-problem/percona1.png)

上图中 963 行是获取 `SHOW ENGINE INNODB STATUS;` 结果中的 `insert buffer` 相关数据，保存在 `$result` 这个数组中，对于 MySQL 5.5 这个方法已经无效了，因此将新脚本中的方法插入进来：

![](/img/in-post/cacti-problem/percona2.png)

上图中 956 ~ 962 行为从新版本的脚本中提取出的代码段，插入到老脚本的对应位置，就可以获取到 MySQL 5.5 版本的 `insert buffer` 相关数据了，注意段代码还在此函数中加入了一个新变量 `$prev_line` 用来在迭代时保存上一次迭代的输出

将 `$prev_line` 这个新变量也加入到老版本的脚本中对应位置：

![](/img/in-post/cacti-problem/percona3.png)

在这个函数的最后，加入 1098 行 这段代码，这样这个脚本就能够获取到 MySQL 5.5 版本的 Insert Buffer 的数据了。

![](/img/in-post/cacti-problem/cacti_pic4.png)

修改之后，可以看到 InnoDB Insert Buffer  这个监控图已经有数据了。








