---
layout:     post
title:      "MySQL 运行环境优化(Linux)"
subtitle:   "小白的数据库调优指北"
date:       2014-11-14 22:00:00
author:     Liao
catalog:    true
header-img: img/post-bg-2015.jpg
permalink:  /mysql-optimize-in-linux/
tags:
    - MySQL
    - Performance Tuning
    - Operating System
---

数据库系统通常是企业的核心应用，因此针对运行 MySQL 的 Linux 系统通常需要进行一些特殊的优化。

本人非 MySQL 大牛，本文不涉及 MySQL 参数优化，仅对 Linux 系统优化进行一些总结。

# CPU

现代操作系统都是多任务多用户的操作系统，服务器能够支持的CPU数量也越来越多，对于多CPU的服务器有多种架构，常见的是 **NUMA** 和 **SMP**.

**SMP**

SMP 即 Symmetric Multi-Processing，对称多处理，这种架构中，每个处理器的地位都是平等的，对资源的使用权限相同，它们共享同一主存，由一个操作系统控制。

由于共享同一主存，随着 CPU 增多，对内存的资源的争用也会增多，造成 CPU 资源的浪费，因此CPU增多，系统并不能线性提升。实验表明 SMP 服务器CPU 个数为2-4个时 CPU 的利用率最佳。

![](/img/in-post/mysql-optimize-in-linux/smp.gif)

**NUMA**

NUMA, Non-Uniform Memory Access，非一致性内存访问，这种架构下，每个 CPU 都有自己的专用内存和内存控制器（可能就在 CPU 内部），这样一来 CPU 对内存的争用情况将减少。

每个 CPU 都有自己的运行队列，内核会平衡各个内存的运行队列(rebalancing)，当0号CPU的运行进程被调度至1号 CPU 的运行队列中，由于进程需要原来的数据，那么1号 CPU 就有可能需要访问0号 CPU 的专用内存（请求0号CPU的内存控制器），此时需要跨越 CPU 插槽，需要多个 CPU 时钟周期。

在 NUMA 结构下，应该尽可能的保证 CPU 至访问自身的内存，常用的方法是进程绑定（CPU affinity）。

两种架构的图示：

![](/img/in-post/mysql-optimize-in-linux/smp-numa.png)

NUMA 架构不适合用于运行数据库服务，由于每个 CPU 使用自身的内存，因此可能出现一个 CPU 的内存使用完，而另一个 CPU 的内存有大量剩余，造成内存资源的浪费，甚至可能出现内存有较多剩余，系统却一直使用 swap 空间的情况（具体可参见 [MySQL "swap insanity"][mysql-swap-insanity]）。

NUMA 架构的内存分配方式可以设置，但最简单的方式是关闭 NUMA 特性。

可以使用 `numastat` 或 `numastat -p <PID>` 查看 NUMA 结构 CPU 内存访问状态

**关闭 NUMA**

1. 从 BIOS 关闭
2. 在操作系统中关闭，在 `/etc/grub/grub.conf` 的 kernel 行追加 `numa=off`
3. 修改 `/etc/init.d/mysql` 或者 `mysqld_safe` 脚本，这种方式较复杂，不便于管理，不推荐
4. 启动 MySQL 的时候，关闭 NUMA 特性，使用 `numactl --interleave=all mysqld &`

最好的方法还是 1 和 2


# Memory

### vm.swappiness

vm.swappiness 是操作系统使用交换分区的策略，它的值从 0 至 100，系统默认值为 60. 值为 0 表示尽量少使用 swap，100 表示尽量将 inactive 的内存页交换出去。当系统内存使用到一定量时，系统就根据这个参数判断是否使用交换分区。

由于使用 swap 会导致数据库性能急剧下降，有人建议不使用 swap，这样是比较危险的，因为当系统出现 OOM(Out Of Memory) 时，系统会启动OOM-killer，有可能杀死 MySQL 进程，造成业务中断，甚至丢失数据。

我建议设置较小的 swap 分区，并设置 `vm.swappiness=0`，不要忘了写进 `/etc/sysctl.conf` 永久生效，swap 分区不宜太大是因为如果 vm.swappiness 设置不当可能造成大量的内存交换。

**注意**：在较新的内核中(2.6.32-303.el6及以后) 对 vm.swappiness 设置为 0 时的系统使用交换分区策略进行了调整，系统永远不会使用交换分区，这意味着如果出现内存耗尽，系统将启动 OOM-killer 杀死消耗内存的进程。因此 可以设置`vm.swappiness=1`.

**再次注意**： Kernel 2.6.28 之前的版本中（如 CentOS 5），有 Bug，不管设置 vm.swappiness 设置为多大，由于内存管理算法的问题，Linux 都有可能会将 MySQL 内存交换至交换分区，如果你经常遇到这种情况，修复的方法是升级你的 kernel，或者考虑使用这个脚本 [A perl script you can run from cron.][perl]


# I/O


### IO scheduler

针对机械磁盘的特性，Linux 引入了多种 IO 调度器来优化 IO 性能，简单来说优化磁盘读写的策略就是将 IO 请求合并与重排。合并操作是将相邻扇区的 IO 请求合并为一个，重排是将 IO 请求按扇区逻辑地址顺序排列。

Linux2.4 只有一种 IO 调度器，2.6后引入了多种 IO 调度器，这里不多介绍，具体可以参考我的另一片文章——[Linux 性能优化之 IO 子系统][IO]

Linux 默认使用 IO 调度算法是 CFQ，该算法可能会出现 IO 请求饿死的情况。

下图为 [Choosing an I/O Scheduler for Red Hat® Enterprise Linux® 4 and the 2.6 Kernel](http://www.redhat.com/magazine/008jun05/features/schedulers/) 中的测试结果

![](/img/in-post/mysql-optimize-in-linux/scheduler.jpg)

从很多实际测试场景来看，Deadline 更适合跑 MySQL 数据库。

修改 IO 调度算法为 Deadline

`echo deadline > /sys/block/sda/queue/scheduler`

这里需要注意，通常的调度算法的行为都是合并请求，排序请求，这些行为是针对机械磁盘的特性来优化的，对于使用 SSD 硬盘的系统，由于没有了磁头寻道，磁片旋转定位等操作，对 SSD 硬盘使用通常的调度算法就变得没有意义，因此我们使用一种特殊的调度算法 NOOP(NO OPeration)，即不对 IO 请求进行操作，直接按 FIFO 规则进行处理。

设置调度算法为 NOOP.

`echo noop > /sys/block/sda/queue/scheduler`


### Filesystem

现代操作系统都使用日志型文件系统，Linux 中一般使用 ext4 或者 xfs，两者的具体性能比较我不太清楚，据说 XFS 更适合大文件 IO.

文件系统的挂载参数，建议使用 **noatime**，**nobarrier** 两个选项。

**noatime**

默认情况下，文件每次被读取或修改（修改也需要先读取），系统将更新atime并写入至文件元数据中。

使用 noatime 挂载文件系统，访问文件时将不对文件的 atime 进行更新，此时 atime 就相当于 mtime。磁盘性能可以得到 0-10% 的提升。

大多数时候关闭文件的 atime 也只能获得较小的性能提升（这个说法引用于 IMB Redbook），但是也聊胜于无，在 `/etc/fstab` 这样挂载

`/dev/sdb /mountlocation ext4 	default, noatime 0 0`

**nobarrier**

barrier 是保证日志型文件系统的 WAL(Write Ahead Logging) 的一种手段，数据写入磁盘时，理应先写入 journal 区，再写入数据在磁盘的实际对应位置，磁盘厂商为了加快磁盘写入速度，磁盘都会内置 cache，数据一般都先写入磁盘的 cache.

cache 能加快磁盘写入速度，但磁盘一般会对 cache 内缓存数据排序使之按最优的数序刷写到磁盘，这样就可能导致要刷写的数据和实际 journal 记录顺序错乱。一旦系统崩溃，下次开机时磁盘要参考 journal 区来恢复数据，但此时 journal 记录顺序与数据实际刷写顺序不同，就会导致数据反而「恢复」到不一致状态了。而 barrier 如其名 —— 栅栏，先加一个「栅栏」，保证 journal 总是先写入记录，然后对应数据才刷写到磁盘，这种方式保证了系统崩溃后磁盘恢复的正确性，但对写入性能有影响。

数据库服务器底层存储设备要么采用 RAID 卡，RAID 卡本身的电池可以掉电保护；要么采用 Flash 卡，它也有自我保护机制，保证数据不会丢失。所以我们可以安全的使用 **nobarrier** 挂载文件系统。设置方法如下：

对于 ext3，ext4 和 reiserfs 文件系统可以在 mount 时指定 `barrier=0`；对于xfs可以指定 `nobarrier` 选项。

# 总结

- CPU 方面，禁用 NUMA
- 内存方面，设置 vm.swappiness=0
- IO 方面，使用 deadline 或者 noop 调度策略，使用 noatime，nobarrier 挂载文件系统

</br>
深入了解 MySQL 在操作系统方面的调优，可以阅读这个 PDF —— [Linux and H/W optimizations for MySQL][pdf]

还有一些能对 MySQL 运行情况进行一些简单判断和分析的脚本，可以阅读这个了解 —— [MySQL Performance Tuning Scripts and Know-How][scripts]

</br>

参考：

- [LINUX上MYSQL优化三板斧](http://www.woqutech.com/?p=1200)
- [优化Mysql的运行环境(Linux)](http://get.jobdeer.com/910.get)
- [Linux performance tuning tips for MySQL](http://www.percona.com/blog/2013/12/07/linux-performance-tuning-tips-mysql/)
- [4 performance fixes to MySQL on large servers](http://openlife.cc/blogs/2011/may/4-performance-fixes-mysql-large-servers)

</br></br>

[mysql-swap-insanity]: http://blog.jcole.us/2010/09/28/mysql-swap-insanity-and-the-numa-architecture/ "swap insanity"
[perl]: https://github.com/yoshinorim/unmap_mysql_logs
[pdf]: http://en.oreilly.com/mysql2011/public/schedule/detail/17111
[scripts]: http://www.askapache.com/mysql/performance-tuning-mysql.html
[IO]: http://liaoph.com/linux-system-io/#io




