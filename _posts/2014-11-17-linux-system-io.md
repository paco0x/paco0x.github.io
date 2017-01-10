---
layout:     post
title:      "Linux 性能优化之 IO 子系统"
subtitle:   "Dive into IO subsystem"
date:       2014-11-17 23:51:00
author:     Liao
catalog:    true
header-img: img/post-bg-2015.jpg
permalink:  /linux-system-io/
tags:
    - Performance Tuning
---

本文介绍了对 Linux IO 子系统性能进行优化时需要考虑的因素，以及一些 IO 性能检测工具。

本文的大部分内容来自 IBM Redbook - [Linux Performance and Tuning Guidelines][IBM]


# FileSystem


## VFS(Virtual FileSystem) 虚拟文件系统

文件系统是内核的功能，是一种工作在内核空间的软件，访问一个文件必须要需要文件系统的存在才可以。Linux 可以支持多达数十种不同的文件系统，它们的实现各不相同，因此 Linux 内核向用户空间提供了虚拟文件系统这个统一的接口用来对文件系统进行操作。

![](/img/in-post/linux-system-io/vfs.png)

虚拟文件系统是位于用户空间进程和内核空间中多种不同的底层文件系统的实现之间的一个抽象的接口层，它提供了常见的文件系统对象模型（如 i-node, file object, page cache, directory entry, etc.）和访问这些对象的方法（如 open, close, delete, write, read, create, fstat, etc.），并将它们统一输出，类似于库的作用。从而向用户进程隐藏了各种不同的文件系统的具体实现，这样上层软件只需要和 VFS 进行交互而不必关系底层的文件系统，简化了软件的开发，也使得 linux 可以支持多种不同的文件系统。


### Journaling

**非日志型文件系统**

在非日志型文件系统中，对文件系统实施一个写操作，内核会首先修改对应的元数据，然后修改数据块。如果在写入元数据时，文件系统发生崩溃或某种故障，那么数据的一致性将会遭到破坏。fsck 命令可以在下次重启时检查所有的元数据并修复数据一致性，但是如果文件系统非常大，或者系统运行关键业务不允许停机，使用非日志型文件系统的风险会非常高。

**日志型文件系统**

日志型文件系统的区别在于，在进行对文件系统写数据之前，写将数据写到「日志区」，然后再写入文件系统，在写入文件系统之后删除日志。日志区可以在文件系统内部也可以在文件系统外部。日志区的数据称作文件系统日志，这些数据包含了修改了的元数据，也可能包含将要修改的数据。写日志也会带来一定的额外开销。


## EXT2

![](/img/in-post/linux-system-io/ext2.png)

ext2 文件系统组成不再赘述，需要注意的是 ext2 文件系统没有日志功能。

## EXT3

ext3 是带有日志功能文件系统，它基于ext2文件系统实现。

- 以一致性的方式写入数据，即使断电或文件系统崩溃，恢复的时间会大大减少

-  数据完整性：在挂载时指定选项 `data=journal` 则可以使日志记录下元数据和文件数据

-  速度：指定挂载选项 `data=writeback` 使用回写的方式写数据

-  灵活性：可以从 ext2 系统升级，而不需要重新格式化文件系统，也可以以非日志模式挂载，就如同 ext2 一样

**ext3的日志模式**

- *journal*，提供最大数据完整性，日志会记录元数据和文件数据

- *ordered*，日志只记录元数据。但是会保证文件数据先被写入，这是默认选项。

- *writeback*，使用回写的方式写数据，只保证记录元数据至日志。
	
**其他文件系统**

- *ReiserFS*，ReiserFS是一个快速的日志型文件系统，有着良好的磁盘空间使用和快速的崩溃恢复。属于 Novell 公司，在 SUSE Linux 中使用。

- *JFS (Journal File System)*， JFS 是一个完全 64 位文件系统，可以支持非常大的文件和分区，早先由 IBM 公司为 AIX 操作系统开发，现已使用 GPL 协议开源。JFS适用于非常大的分区和文件如 HPC 或者数据库业务。

- *XFS (eXtended File System)*， 一个高性能的日志型文件系统，和 JFS 比较相似。
 

# I/O子系统架构

![](/img/in-post/linux-system-io/io-arch.png)

上图概括了一次磁盘 write 操作的过程，假设文件已经被从磁盘中读入了 page cache 中

1.  一个用户进程通过 write() 系统调用发起写请求

2.  内核更新对应的 page cache

3.  pdflush 内核线程将 page cache 写入至磁盘中

4.  文件系统层将每一个 block buffer 存放为一个 bio 结构体，并向块设备层提交一个写请求

5.  块设备层从上层接受到请求，执行 IO 调度操作，并将请求放入IO 请求队列中

6.  设备驱动（如 SCSI 或其他设备驱动）完成写操作

7.  磁盘设备固件执行对应的硬件操作，如磁盘的旋转，寻道等，数据被写入到磁盘扇区中


## Block Layer

Block layer 处理所有和块设备相关的操作。block layer 最关键是数据结构是 bio 结构体。bio 结构体是 file system layer 到 block layer 的接口。
当执行一个写操作时，文件系统层将数据写入 page cache（由 block buffer 组成），将连续的块放到一起，组成 bio 结构体，然后将 bio 送至  block layer。

block layer 处理 bio 请求，并将这些请求链接成一个队列，称作 IO 请求队列，这个连接的操作就称作 IO 调度（也叫 IO elevator 即电梯算法）. 

<h2 id="io">IO scheduler</h2>

IO 调度器的总体目标是减少磁盘的寻道时间（因此调度器都是针对机械硬盘进行优化的），IO 调度器通过两种方式来减少磁盘寻道：**合并**和**排序**。

合并即当两个或多个 IO 请求的是相邻的磁盘扇区，那么就将这些请求合并为一个请求。通过合并请求，多个 IO 请求只需要向磁盘发送一个请求指令，减少了磁盘的开销。

排序就是将不能合并的 IO 请求，根据请求磁盘扇区的顺序，在请求队列中进行排序，使得磁头可以按照磁盘的旋转顺序的完成 IO 操作，可以减小磁盘的寻道次数。

调度器的算法和电梯运行的策略相似，因此 IO 调度器也被称作 IO 电梯( IO Elevator )。由于对请求进行了重排，一部分的请求可能会被延迟，以提升整体的性能。

Linux 2.4 只使用了一种通用的 IO 算法。到 Linux 2.6 实现了 4 种 IO 调度模型，其中 *anticipatory* 在 2.6.33 中被**移除**。

### Linus Elevator

早先的 IO 调度器就叫做 Linus Elevator，当一个请求被加入到请求队列中，它首先检查队列中是否存在相邻的请求以合并两个请求，这可能包含前合并和后合并。如果不能合并，则寻找是否能够将新请求按扇区顺序插入到请求队列，如果没有找到适合插入的位置，那么就将这个请求插入到队列的末尾。另外，如果请求队列中的某个请求超过的预先定义的过期阈值，新请求即使可以进行排序，也被插入到队列的末尾。这样可以防止磁盘上某个区域产生大量请求，而其他区域的请求被饿死。然而，这种过期策略并不十分高效。

这种算法可能导致请求饿死的情况，它是 Linux 2.4 的唯一调度器。

当一个请求加入到请求队列时，IO 调度器所完成的操作如下

1. 如果队列中存在对相邻扇区的请求，则合并两个请求为一个

2. 如果队列中存在超过过期时间的请求，那么新请求被插入到队列的末尾，以防止饿死老的请求

3. 如果队列中存在可以按扇区地址排序的合适位置，那么将请求插入到这个位置

4. 如果队列中没有合适的可插入位置，请求被插入到队列末尾


### Deadline - latency-oriented

Deadline 调度器是设计用来解决 Linus Elevator 导致的 I/O 饿死的问题。对磁盘上某个区域的大量请求，会无限期的饿死 (starvation) 对磁盘其他区域上的请求。

请求饿死的一个特例是 **写请求饿死读请求** (writes starving reads)，对于进程来说，当需要进行写操作时，由于需要写的数据在内存的 page cache 中，进程是需要修改对应的内存，向内核发送写请求即可，之后进程可以去进行其他操作，由内核负责将数据同步至磁盘中即可。对于进程来说，写请求是异步操作。而读操作是完全不同的，当进程需要读取数据时，需要向内核发送读请求，内核将数据载入到内存中，由于进程往往需要处理读取的数据，因此进程将处于阻塞状态，直到请求被处理完成。对于进程来说，读请求是同步的操作。写请求延迟对系统的性能影响不大，而读请求延迟对系统性能影响是非常大的，因此两种 IO 请求需要区别对待。

Dealine 调度器专门针对读请求延迟进行了优化，在 deadline 算法中，每一个请求都有一个过期时间。默认情况下，读请求的过期时间是 500ms，写请求的过期时间是 5s。Dealine 调度器也会对请求队列进行合并和排序操作，这个队列称作排序队列（sorted queue）。当新请求被提交，Deadline将其加入到排序队列中进行合并和排序。同时 Deadline 将这个请求加入到第二种类型的队列中，读请求被加入至读FIFO队列 (Read FIFO queue)，写请求被加入到写FIFO队列 (Write FIFO queue)，这两个队列中，请求完全按照 FIFO 顺序排列，即新请求永远被放入到队列的末尾。

这样一来 Dealine 就维护三个队列，正常情况下，Deadline 将排序队列中的请求放入到调度队列 (dispatch queue，即将写入磁盘的队列)中，调度队列把请求发送至磁盘驱动。如果写 FIFO 队列或读 FIFO 队列中的请求发生了超时，Deadline 调度器就不再使用排序队列，而是开始将发生超时的 FIFO 队列的请求放入调度队列，直至队列中没有超时的请求，Deadline 通过这样的方式保证所有的请求都不会长时间超时。

![](/img/in-post/linux-system-io/deadline.gif)

Deadling 防止了请求饿死的出现，由于读请求的超时时间远小于写请求，它同时也避免了出现写请求饿死读请求。

Deadline 比较适合于MySQL数据库。


### Anticipatory(AS)

AS 调度器是基于 Deadline 调度器，加上了一个启发式的「预测」，假设系统中有大量的写请求，这时如果夹杂几个读请求，由于读请求的过期时间短，读请求立即在多个写请求的中间产生，这样会导致磁盘的来回寻道，AS 试图减少大量写请求夹杂少量读请求产生的寻道风暴（seek storm）。当一个读请求完成后，AS不会立即返回处理队列中的剩余请求，而是等待一个预测时间（默认为 6ms），如果等待的时间内发生了相邻位置的读请求，那么立即处理这个相邻位置的读请求，再返回处理队列中的请求，这样可以优化多余的寻道时间。

它可以为连续 IO 请求（如顺序读）进行了一定的优化，但是对于随机读的场景 AS 会产生较大的延迟，对于数据库应用很糟糕，而对于 Web Server 可能会表现的不错。这个算法也可以简单理解为面向低速磁盘的，对于使用了 TCG（Tagged Command Queueing）高性能的磁盘和 RAID，使用 AS 会降低性能。

在 Linux 2.6 - 2.6.18 AS 是内核默认调度器，然而大多数的企业发行版选择的是 CFQ 调度器。

到Linux 2.6.33 版本，AS 被从内核中移除，因为可以通过调整其他调度器（如 CFQ）来实现与其相似的功能。


### Complete Fair Queuing（CFQ）- fairness-oriented

CFQ 为每个进程分配一个 I/O 请求队列，在每个队列的内部，进行合并和排序的优化。CFQ 以轮询的方式处理这些队列，每次从一个队列中处理特定数量的请求（默认为 4 个）。它试图将 I/O 带宽均匀的分配至每个进程。CFQ 原本针对的是多媒体或桌面应用场景，然而，CFQ 其实在许多场景中内表现的很好。CFQ  对每一个 IO 请求都是公平的。这使得 CFQ 很适合离散读的应用 (eg: OLTP DB)

多数企业发型版选择 CFQ 作为默认的 I/O 调度器。

 
### NOOP(No Operation)

该算法实现了最简单的 FIFO 队列，所有 IO 请求按照大致的先后顺序进行操作。之所以说「大致」，原因是 NOOP 在 FIFO 的基础上还做了相邻 IO 请求的合并，但其不会进行排序操作。 

NOOP 适用于底层硬件自身就具有很强调度控制器的块设备（如某些SAN设备），或者完全随机访问的块设备（如SSD）。

各个调度器在数据库应用下的性能，可以看出CFQ的性能是比较均衡的

![](/img/in-post/linux-system-io/scheduler-db.jpg)

# IO 参数调整

## 队列长度

查看队列长度 </br>
`/sys/block/<dev>/queue/nr_requests`

下图展示了各种队列长度时，Deadline 和 CFQ 调度器的性能

![](/img/in-post/linux-system-io/queue-1.png)
![](/img/in-post/linux-system-io/queue-2.png)

由 ext3 的表现可以看出，对于大量的小文件写操作，队列长度更长，性能会有所提升，在 16KB 左右，性能提升最为明显，在 64KB 时，64 至 8192 的队列长度有着差不多的性能。随着文件大小的增大，小队列长度反而有着更好的性能。
RHEL 操作系统中，每个设备有一个队列长度。对于类似数据库日志的存放分区，大部分写操作属于小文件 IO，可以将队列长度调小。

对于大量的连续读取，可以考虑增加读取首部的窗口大小</br>
`/sys/block/<dev>/queue/read_ahead_kb`

## IO调度器

选择设备的IO调度器 </br>
`/sys/block/<dev>/queue/scheduler`

或者在 grub.conf 中加入内核参数 `elevator=SCHEDULER`

### Deadline参数

`/sys/block/<device>/queue/iosched/writes_starved` </br>
进行一个写操作之前，允许进行多少次读操作

`/sys/block/<device>/queue/iosched/read_expire` </br>
读请求的过期时间

`/sys/block/<device>/queue/iosched/read_expire` </br>
写请求的过期时间，默认为 500ms

`/sys/block/sda/queue/iosched/front_merges` </br>
是否进行前合并

### Anticipatory参数

`/sys/block/<device>/queue/iosched/antic_expire` </br>
预测等待时长，默认为 6ms

`/sys/block/<device>/queue/iosched/{write_expire,read_expire}` </br>
读写请求的超时时长

`/sys/block/<device>/queue/iosched/{write_batch_expire,read_batch_expire}` </br>
读写的批量处理时长
	
### CFQ参数

`/sys/block/<device>/queue/iosched/slice_idle` </br>
当一个进程的队列被分配到时间片却没有 IO 请求时，调度器在轮询至下一个队列之前的等待时间，以提升 IO 的局部性，对于 SSD 设备，可以将这个值设为 0。

`/sys/block/<device>/queue/iosched/quantum` </br>
一个进程的队列每次被处理 IO 请求的最大数量，默认为 4，RHEL6 为 8，增大这个值可以提升并行处理 IO 的性能，但可能会造成某些 IO 延迟问题。

`/sys/block/<device>/queue/iosched/slice_async_rq` </br>
一次处理写请求的最大数

`/sys/block/<device>/queue/iosched/low_latency` </br>
如果IO延迟的问题很严重，将这个值设为 1
	
**调整CFQ调度器的进程IO优先级**

`$ ionice [[-c class] [-n classdata] [-t]] -p PID [PID]...`
`$ ionice [-c class] [-n classdata] [-t] COMMAND [ARG]...`

CFQ 的进程 IO 优先级和进程 CPU 优先级是独立的。使用 ionice 调整进程优先级，有三种调度类型可以选择

- **idle** </br>
只有在没有更高优先级的进程产生 IO 时，idle 才可以使用磁盘 IO，适用于哪些不重要的程序（如 updatedb），让它们在磁盘空闲时再产生 IO

- **Best-effort** </br>
这个类型共有 8 个优先级，分别为 0-7，数字越低，优先级越高，相同级别的优先级使用轮询的方式处理。适用于普通的进程。</br> </br>
在2.6.26之前，没有指定调度类型的进程使用"none" 调度类型，IO调度器将其视作Best-effort进行处理，这个类别中进程优先级由CPU优先级计算得来: io_priority = (cpu_nice + 20) / 5 </br>		
2.6.26之后，没有指定调度类型的进程将从CPU调度类型继承而来，这个类别的优先级仍按照CPU优先级计算而来: io_priority = (cpu_nice + 20) / 5
		
- **Real time** </br>
这个调度级别的进程产生的IO被优先处理，这个调度类型应小心使用，防止饿死其他进程IO, 它也有8个优先级，数字越大分的的IO时间片越长


## 文件系统

一般来说 ReiserFS 更适合于小文件 IO，而 XFS 和 JFS 适合大文件 IO，ext4 则处于两种之间。

JFS 和 XFS 适用于大型的数据仓库，科学计算，大型 SMP 服务器，多媒体流服务等。ReiserFS 和 Ext4 适合于文件服务器，Web 服务器或邮件服务器。

### noatime

atime 用于记录文件最后一次被读取的时间。默认情况下，文件每次被读取或修改（也需要先读取），系统将更新 atime 并写入至文件元数据中。由于写操作是很昂贵的，减少不必要的写操作可以提升磁盘性能。然后，大多数时候，关闭文件的 atime 也只能获得非常小的性能提升（这个说法来自于IBM Redbook，表示怀疑）。

使用 noatime 挂载文件系统，将不对文件的 atime 进行更新，此时 atime就相当于 mtime。磁盘性能可以得到0-10%的提升。

使用 noatime挂载方法  </br>
`/dev/sdb1 /mountlocation ext3 defaults,noatime 1 2`

### nobarrier

barrier 是保证日志文件系统的 WAL (write ahead logging) 一种手段，数据写入磁盘时，理应先写入 journal 区，再写入数据在磁盘的实际对应位置，磁盘厂商为了加快磁盘写入速度，磁盘都内置 cache，数据一般都先写入磁盘的 cache。
 
cache 能加快写入速度，但磁盘一般会对 cache 内缓存数据排序使之最优刷新到磁盘，这样就可能导致要刷新的实际数据和 journal 顺序错乱。一旦系统崩溃，下次开机时磁盘要参考 journal 区来恢复，但此时 journal 记录顺序与数据实际刷新顺序不同就会导致数据反而「恢复」到不一致了。而barrier 如其名——「栅栏」，先加一个「栅栏」，保证 journal 总是先写入记录，然后对应数据才刷新到磁盘，这种方式保证了系统崩溃后磁盘恢复的正确性，但对写入性能有影响。

数据库服务器底层存储设备要么采用 RAID 卡，RAID 卡本身的电池可以掉电保护；要么采用 Flash 卡，它也有自我保护机制，保证数据不会丢失。所以我们可以安全的使用 nobarrier 挂载文件系统。设置方法如下： 
对于 ext3，ext4 和 reiserfs 文件系统可以在 mount 时指定 barrier=0；对于 xfs 可以指定 nobarrier 选项。

### read-ahead 预读

Linux 把读模式分为随机读和顺序读两大类，并只对顺序读进行预读。这一原则相对保守，但是可以保证很高的预读命中率。

为了保证预读命中率，Linux只对顺序读 (sequential read) 进行预读。内核通过验证如下两个条件来判定一个 read() 是否顺序读：

1. 这是文件被打开后的第一次读，并且读的是文件首部；
2. 当前的读请求与前一（记录的）读请求在文件内的位置是连续的。

如果不满足上述顺序性条件，就判定为随机读。任何一个随机读都将终止当前的顺序序列，从而终止预读行为（而不是缩减预读大小）。

#### 预读窗口

Linux采用了一个快速的窗口扩张过程

- 首次预读： `readahead_size = read_size * 2  // or *4` </br>
预读窗口的初始值是读大小的二到四倍。这意味着在您的程序中使用较大的读粒度（比如32KB）可以稍稍提升I/O效率。

- 后续预读：`readahead_size *= 2` </br>
后续的预读窗口将逐次倍增，直到达到系统设定的最大预读大小，其缺省值是 256KB
	
查看和修改预读窗口 </br>
`$ blockdev -getra device` </br>
`$ blockdev -setra N device`

	
### 日志模式

大多数文件系统可以设置三种日志模式，对于 ext4 文件系统，日志模式对磁盘的性能有较大的影响。

- `data=journal` </br>
数据和元数据都写入日志，提供了最高的数据一致性

- `data=ordered` (默认) </br>
只记录元数据，然后它会保证先将数据写入磁盘

- `data=writeback` </br>
采用回写的方式，牺牲数据一致性，获得更好的性能。仍然会将元数据记录到日志中，此模式对小文件 IO 性能提升最为明显，但可能造成数据丢失。
	
**将日志放在单独的设备中**

1. 卸载文件系统

2. 查看文件系统块大小和日志参数 </br>
	`$ dumpe2fs /dev/sdb1`

3. 移除文件系统内部的日志区 </br>
	`$ tune2fs  -O  ^has_journal  /dev/sdb1`

4. 创建外部的日志设备 </br>
	`$ mke2fs –O  journal_dev  -b  block-size  /dev/sdc1`

5. 更新原文件系统的超级块 </br>
	`$ tune2fs  -j  -J  device=/dev/sdc1  /dev/sdb1`

**commit**

设置多少秒从日志中进行一个同步，默认是5


### 优化Ext4

1. 格式大文件系统化时延迟 inode 的写入 </br></br>
对于超大文件系统，mkfs.ext4 进程要花很长时间初始化文件系统中到所有内节点表。 </br></br>
可使用 `-E lazy_itable_init=1` 选项延迟这个进程。如果使用这个选项，内核进程将在挂载文件系统后继续初始化该文件它。 </br>
`$ mkfs.ext4 -E lazy_itable_init=1 /dev/sda5`

2. 关闭Ext4的自动 fsync() 调用 </br>
`-o noauto_da_alloc` 或 `mount -o noauto_da_alloc`

3. 降低日志IO的优先级至与数据IO相同 </br>
`-o journal_ioprio=n`  </br>
n的用效取值范围为0-7，默认为3；


### 查看文件系统锁
`$ cat /proc/locks`



# Benchmark 基准测试

### iozone

使用iozone对文件系统进行基准测试

- `$ iozone -a` </br>
iozone将在所有模式下进行测试，使用记录块从4k到16M，测试文件大小从64k到512M

- `$ iozone -Ra` 或 `iozone -Rab output.xls` </br>
iozone将测试结果放在Excel中

- `$ iozone -Ra -g 2g` </br>
如果内存大于512MB，则测试文件需要更大；最好测试文件是内存的两倍。例如内存为1G，将测试文件设置最大为2G

`$ iozone -Ra -g 2g -i 0 -i 1` </br>
如果只关心文件磁盘的read/write性能，而不必花费时间在其他模式上测试，则我们需要指定测试模式
 
`$ iozone -Rac` </br>
如果测试NFS，将使用-c，这通知iozone在测试过程中执行close()函数。使用close()将减少NFS客户端缓存的影响。但是如果测试文件比内存大，就没有必要使用参数-c

```plain
-a	在所有模式下进行测试，使用记录块从4k到16M，测试文件大小从64k到512M
-b	生成excel文件
-R	生成excel输入
-f	指定临时文件名
-g	自动模式的最大文件大小
-n	自动模式的最小文件大小
-s	指定文件大小，默认单位是KB, 也可以使用m 和 g
-y	自动模式的最小记录大小，默认单位是KB
-q	自动模式的最大记录大小，默认单位是KB
-r	指定记录大小，单位是KB
-i	选择测试模式
```

```
0=write/rewrite
1=read/re-read
2=random-read/write
3=Read-backwards
4=Re-write-record
5=stride-read
6=fwrite/re-fwrite
7=fread/Re-fread,
8=random mix
9=pwrite/Re-pwrite
10=pread/Re-pread
11=pwritev/Re-pwritev
12=preadv/Repreadv
```

一个例子

    $ iozone -a -s 128M -y 512 -q 16384 -i 0 -i 1 -i 2 -f /test/a.out -Rb /root/ext3_writeback.out


### dd

`$ dd bs=1M count=128 if=/dev/zero of=test conv=fdatasync` </br>
使用这个参数，dd命令执行到最后会真正执行一次同步(sync)操作
	
`$ dd bs=1M count=128 if=/dev/zero of=test oflag=direct`
使用直接IO（绕过缓存）

### bonnie++

```
usage: bonnie++ [-d scratch-dir] [-s size(Mb)[:chunk-size(b)]]
 [-n number-to-stat[:max-size[:min-size][:num-directories]]]
 [-m machine-name]
 [-r ram-size-in-Mb]
 [-x number-of-tests] [-u uid-to-use:gid-to-use] [-g gid-to-use]
 [-q] [-f] [-b] [-p processes | -y]
```
 
```
-d 生成测试文件的路径
-s 生成测试文件的大小，以M为单位（如果不使用-r参数，则要求文件大小至少是系统物理内存的2倍）
-m 机器名，实际上我们可以认为是本次测试的方案名，可以随便定义。默认是本机的hostname。
-r 内存大小，指定内存大小，这样可以通过-s参数创建r*2大小的文件，通常用于缩短测试时间，但是需要注意这样由于内存的cache可能导致测试结果的不准确
-x 测试的次数
-u 测试文件的属主和组，默认是执行bonnie++的当前用户和当前组
-g 测试文件的组，默认是执行bonnie++的当前用组
-b 在每次写文件时调用fsync()函数，对于测试邮件服务器或者数据库服务器这种通常需要同步操作的情况比较适合，而不使用该参数则比较适合测试copy文件或者编译等操作的效率。
```
 
可以简单地运行如下命令进行磁盘性能测试：

`$ bonnie++ -d /global/oradata –m sun3510` </br>
这样将会在指定的目录下（通常我们会指定一个盘阵上卷的挂载点），生成相当于主机物理内存两倍的文件，如果总量大于1G，则生成多个大小为1G的文件。假设主机内存为4G，那么在测试中就会生成8个1G的文件，到测试结束，这些文件会被自动删除。
 
如果我们的主机内存是4G，但是我们想缩短测试的时间，比如说只写2G的文件，就应该执行下面的命令：

`$ bonnie++ -d /global/oradata –m sun3510 –s 2048 –r 1024` 
 


### blktrace & blkparse
 
blktrace 是一个针对 Linux 内核中块设备 IO 层的跟踪工具，用来收集磁盘IO 信息中当 IO 进行到块设备层（block层，所以叫blk trace）时的详细信息（如 IO 请求提交，入队，合并，完成等等一些列的信息），blktrace 需要借助内核经由 debugfs 文件系统（debugf s文件系统在内存中）来输出信息，所以用 blktrace 工具之前需要先挂载 debugfs 文件系统，blktrace需要结合 blkparse 来使用，由 blkparse 来解析 blktrace 产生的特定格式的二进制数据。

blktrace语法：


     blktrace -d dev [ -r debugfs_path ] [ -o output ] [-k ] [ -w time ] [ -a action ] [ -A action_mask ] [ -v ]
    
    blktrace选项：
     -A hex-mask		#设置过滤信息mask成十六进制mask
     -a mask			#添加mask到当前的过滤器
     -b size			#指定缓存大小for提取的结果，默认为512KB
     -d dev			#添加一个设备追踪
     -I file			#Adds the devices found in file as devices to trace
     -k				#杀掉正在运行的追踪
     -n num-sub		#指定缓冲池大小，默认为4个子缓冲区 
     -o file			#指定输出文件的名字
     -r rel-path		#指定的debugfs挂载点
     -V				#版本号 
     -w seconds		#设置运行的时间


文件输出

`$ blktrace –d /dev/sda –o trace`

解析结果
`$ blkparse -i trace -o /root/trace.txt`

### FIO

FIO 是测试 IOPS 的非常好的工具，用来对硬件进行压力测试和验证，支持 13 种不同的 IO 引擎，包括：sync, mmap, libaio, posixaio, SG v3, splice, null, network, syslet, guasi, solarisaio 等等。 

**随机读** 

    $ fio -filename=/dev/sdb1 -direct=1 -iodepth 1 -thread -rw=randread -ioengine=psync -bs=16k -size=200G 
    -numjobs=10 -runtime=1000 -group_reporting -name=mytest

说明

```
filename=/dev/sdb1       测试文件名称，通常选择需要测试的盘的data目录。 
direct=1                 测试过程绕过机器自带的buffer。使测试结果更真实。 
rw=randwrite             测试随机写的I/O 
rw=randrw                测试随机写和读的I/O 
bs=16k                   单次io的块文件大小为16k 
bsrange=512-2048         同上，提定数据块的大小范围 
size=5g                  本次的测试文件大小为5g，以每次4k的io进行测试。 
numjobs=30               本次的测试线程为30. 
runtime=1000             测试时间为1000秒，如果不写则一直将5g文件分4k每次写完为止。 
ioengine=psync           io引擎使用pync方式 
rwmixwrite=30            在混合读写的模式下，写占30% 
group_reporting          关于显示结果的，汇总每个进程的信息。 
此外 
lockmem=1g               只使用1g内存进行测试。 
zero_buffers             用0初始化系统buffer。 
nrfiles=8                每个进程生成文件的数量。 
```

**顺序读**

    $ fio -filename=/dev/sdb1 -direct=1 -iodepth 1 -thread -rw=read -ioengine=psync -bs=16k -size=200G -numjobs=30 -runtime=1000 -group_reporting -name=mytest


**随机写 **

    $ fio -filename=/dev/sdb1 -direct=1 -iodepth 1 -thread -rw=randwrite -ioengine=psync -bs=16k -size=200G -numjobs=30 -runtime=1000 -group_reporting -name=mytest



**顺序写**

    $ fio -filename=/dev/sdb1 -direct=1 -iodepth 1 -thread -rw=write -ioengine=psync -bs=16k -size=200G -numjobs=30 -runtime=1000 -group_reporting -name=mytest


**混合随机读写**

    $ fio -filename=/dev/sdb1 -direct=1 -iodepth 1 -thread -rw=randrw -rwmixread=70 -ioengine=psync -bs=16k -size=200G -numjobs=30 -runtime=100 -group_reporting -name=mytest -ioscheduler=noop 


### stress

它可以给我们的系统施加 CPU，内存，IO 和磁盘的压力，在模拟极端场景给应用系统造成的压力方面很有帮助。

示例 </br>
`$ stress --cpu 8 --io 4 --vm 2 --vm-bytes 128M --timeout 10d`


</br>
参考：

- [Linux Performance and Tuning Guidelines][IBM]
- [Red Hat Enterprise Linux 6 Performance Tuning Guide](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/6/html-single/Performance_Tuning_Guide/index.html)
- [Choosing an I/O Scheduler for Red Hat® Enterprise Linux® 4 and the 2.6 Kernel](http://www.redhat.com/magazine/008jun05/features/schedulers/)
- [RHEL 的 I/O 调度器(Scheduler)与 Database 的关系](http://dbanotes.net/database/rhel_io_scheduler_database.html)
[I/O Schedulers](http://www.makelinux.net/books/lkd2/ch13lev1sec5)

</br></br>

[IBM]: http://www.redbooks.ibm.com/abstracts/redp4285.html?Open
