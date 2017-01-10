---
layout:     post
title:      计算机原理 —— 主板与内存映射
subtitle:   "Motherboard and memory map"
date:       2015-04-29 23:00:00
author:     Liao
catalog:    true
header-img: img/post-bg-2015.jpg
permalink:  /motherboard-and-memory-map/
tags:
    - Operating System
---

> 这是来自于 [Gustavo Duarte](http://duartes.org/gustavo/blog/) 博客的一系列的计算机内部原理的科普文章，本人出于兴趣翻译过来。原文：[Motherboard Chipsets and the Memory Map](http://duartes.org/gustavo/blog/post/motherboard-chipsets-memory-map)

我打算写几篇关于计算机内部原理的文章，来帮助解释现代操作系统内核是如何工作的。我希望这些文章能对那些对这部分内容感兴趣但又没有相关经验的爱好者和程序员们有所帮助。文章主要关注 Linux，Windows 和 Intel 处理器。我对内部原理有强烈的爱好，我曾经写过一些内核模式的代码。这是第一篇文章，主要描述现代基于 Intel 处理器的主板架构，CPU 是怎样访问内存的，以及系统的内存映射。

开始之前，我们先来看看一台基于 Intel CPU 的计算机是如何组成的。下面这幅图展示了一块主板的主要组成元素。

![](/img/in-post/motherboard-and-memory-map/motherboard.png)

上图中需要注意的是，CPU 本身并不知道它与什么设备连接在一起。它通过它的[针脚（pins）](http://en.wikipedia.org/wiki/Image:Intel_80486DX2_bottom.jpg) 与外界通信，但是它并不关心外面是谁在与它通信。与之相连的可能是计算机的主板，但也可以是一台烤面包机，路由器，或者其他什么。CPU 主要通过三种方式与外界通信：内存地址空间，I/O 地址空间和中断。我们现在只讨论关于主板和内存的内容。

在主板中，CPU 与外界的唯一通路是将其连接到北桥的前端总线。当 CPU 需要读写内存时，它会使用这条总线。它使用一些针脚用于表示它需要读或者写的物理内存地址。一个 Interl Core 2 QX6600 处理器有 33 个针脚用于传输物理内存地址（因此内存地址可以有 2^33 种变化），64 个针脚用于发送或者接收数据（因此数据以 64 位的带宽进行传输，即 8 字节的块）。这意味着 CPU 最多可以使用 64GB （2^33 * 8 bytes），尽管大多数的芯片只使用 8G 的内存。

接下来重点来了。我们通常都认为内存仅仅指的是 RAM （随机存储内存），RAM 是程序运行后时刻需要与之进行数据存储区。的确大多数 CPU 发出的请求被北桥路由到了 RAM 模块，但并非所有的请求都是这样。物理内存地址还会被用于与主板上各种各样的设备进行通信（这种通信被称作[内存映射 IO （memeory-mapped I/O）](http://en.wikipedia.org/wiki/Memory-mapped_IO)）。这些设备包括显卡，大多数的 PCI 设备，以及用于存储 BIOS 的[闪存](http://en.wikipedia.org/wiki/Flash_memory)。

当北桥收到一个物理内存请求后，它应该决定这个请求被转发到哪里：应该转发至 RAM？还是显卡？这个路由决策是由内存地址映射完成的。对于物理内存地址的每一块区域，内存映射知道这块地址属于哪个设备。大多数的地址被映射到了 RAM 中，如果一个地址没有映射到 RAM，内存映射会告诉芯片此地址上的请求应由哪个设备负责响应。这种内存映射的方式使得 PC 计算机内存的 640KB 至 1MB 的地址中有一个经典的「洞」。当内存地址被用于为显卡和 PCI 设备保留时，会有更大的空隙。这就是为什么 32 位的操作系统[无法使用 4G 的 RAM](http://support.microsoft.com/kb/929605)。Linux 中 `/proc/iomem` 文件列出了这些地址映射范围。下面的图展示了一个 Intel PC 中的前 4G 物理内存地址的典型内存映射。

![](/img/in-post/motherboard-and-memory-map/physical-memory-address.png)

实际的地址范围取决于主板的类型和计算机中存在的设备，但大多数的 Core 2 系统都和上图中描述的类似。所有棕色的区域都被映射到了 RAM 之外的地方。记住这些都是在主板总线上使用的**物理**地址。在 CPU 内部（例如，在程序运行时需要读写的地址），内存地址是**逻辑**的，在访问之前须要先由 CPU 将其转换为对应的物理地址才可以使用的。

将逻辑地址转换为物理地址的规则是很复杂的，同时取决于 CPU 的运行模式（真实模式（real mode），32 位保护模式，64 位保护模式）。不管转换机制是怎样的，CPU 的模式决定了它可以访问多少物理内存。例如，如果 CPU 运行在 32 位模式下，只有 4G 的物理地址空间可用（当然，如果使用了[物理地址扩展](http://en.wikipedia.org/wiki/Physical_address_extension)，情况将不一样，但是现在先忽略它）。因为物理地址最顶端的 1GB 被映射到主板设备，CPU只能使用 3GB 的 RAM（有时可能更少，我有一台 Vista 机器只有 2.4 GB 可用）。如果 CPU 运行在 [真实模式（real mode）](http://en.wikipedia.org/wiki/Real_mode)，那么它只能使用物理 RAM 的 1MB 内容（这是早期 Interl 处理器使用的模式）。另一方面，一个运行在 64 位模式的 CPU 可以访问 64GB （有些芯片还不支持这么多）的物理内存。在 64 位模式下，可以使用总 RAM 之外的物理地址来访问那些 RAM 中被主板设备「偷走」的物理地址。这被称作内存回收，需要芯片的支持。

这就是为了理解下一篇文章需要知道的所有内容，[下一篇文章](http://liaoph.com/how-computers-boot-up/)主要描述从开启电源到 boot loader 到进入内核的启动过程。



















