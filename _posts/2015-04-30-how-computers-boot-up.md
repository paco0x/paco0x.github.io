---
layout:     post
title:      计算机原理 —— 计算机是如何启动的
subtitle:   "How computer booted"
date:       2015-04-30 23:00:00
author:     Liao
catalog:    true
catalog:    true
header-img: img/post-bg-2015.jpg
permalink:  /how-computers-boot-up/
tags:
    - Operating System
---

> 这是来自于 [Gustavo Duarte](http://duartes.org/gustavo/blog/) 博客的一系列的计算机内部原理的科普文章，本人出于兴趣翻译过来。原文：[How Computers Boot Up](http://duartes.org/gustavo/blog/post/how-computers-boot-up/)

上一篇文章讲了 Intel 系统的[主板与内存映射](http://liaoph.com/motherboard-and-memory-map/)。计算机启动是一个复杂的，充满黑科技的（原文是 hacky），多阶段的过程。这里是整个过程的概要图：

![](/img/in-post/how-computers-boot-up/boot-process.png)

当你按下电源键的那一刻，计算机的的启动就开始了。当主板启动后，它开始初始化它自己的固件 —— 芯片组和其他东西 —— 并且试图让 CPU 运行起来。如果这时候出现故障（例如 CPU 坏了或者没有 CPU），那么你看到的可能是一个只有风扇在转的计算机。有一些主板在 CPU 故障时会发出提升声，但是根据我的经验大多数时候只是计算机只会僵死在那里，只有风扇在空转。有时候 USB 或者其他设备也可能导致这种情况发生：拔掉所有非必要的设备可能会解决这种问题。你可以一个个的拔出设备以排查问题原因。

假设 CPU 运行正常。在一个多处理器或者多核系统中，会动态的选择一个 CPU 作为自举处理器（bootstrap processor, BSP），由这颗 CPU 来运行所有的 BIOS 和内核初始化代码。这时候其他的处理器被称作应用程序处理器（application processor, AP），它们会保持停止的状态，直到被内核激活运行。Intel CPU 已经发展了很多年了，但是仍然是完全向后兼容的，因此现代的 CPU 仍可以像古老的 [Intel 8086](http://en.wikipedia.org/wiki/Intel_8086) 那样运行，它们在启动后也确实是这样运行的。在启动状态时，CPU 运行在[真实模式（real mode）](http://en.wikipedia.org/wiki/Real_mode) 下，内存[分页](http://en.wikipedia.org/wiki/Paging)是被关闭的。这就像古老的 MS-DOS 系统一样，只有 1MB 的内存可以被寻址，任何代码都可以对内存的任何地址进行写入 —— 这时还没有保护模式或者权限级别的概念。

CPU 中的大多数的[寄存器](http://en.wikipedia.org/wiki/Processor_register) 在启动后会有预定义好的值，包括指令寄存器（instruction pointer, EIP），它保存有 CPU 执行指令的内存地址。Intel CPU 使用了一个黑科技手段，虽然启动时只有 1MB 的内存可以被寻址，一个隐藏的基地址（本质上，是一个偏移量）被放入了 EIP 中，因此第一条指令的内存地址是 0xFFFFFFF0 （4 GB 内存的最后 16 字节，这显然超出了 1MB 的范围）。这个魔法地址被称作[复位向量（reset vector）](http://en.wikipedia.org/wiki/Reset_vector)，它已经是现代 Intel CPU 的一个标准。

主板确保复位向量的指令是跳转（jump）到映射至 BIOS 入口点的内存地址。这个跳转隐藏了启动时的基地址的存在。主板芯片的[内存映射](http://liaoph.com/motherboard-and-memory-map/)使得这些内存位置都有着与 CPU 指令向匹配的映射内容。跳转后的内存地址被映射到了包含有 BIOS 的闪存中，这时候内存中的还是完全无意义的内容。下面是相关的内存区域示例图：

![](/img/in-post/how-computers-boot-up/memory-map.png)

接着 CPU 开始执行 BIOS 代码，进行一些硬件的初始化。随后 BIOS 进行 [加电自检](http://en.wikipedia.org/wiki/Power_on_self_test)（Power-on Self Test, POST），测试计算机的各个组件。如果缺少显式芯片，POST 会失败，导致 BIOS 停止并发出警告声。如果显式芯片工作正常，屏幕上会打印出制造厂商的 logo，并开始进行内存检测。其他类型的 POST 失败，如缺少键盘，则会导致屏幕上显示错误信息。POST 设计了硬件检测和初始化，包括对资源进行分类 —— 中断，内存范围，I/O 端口等。现代的 BIOS 遵照[高级配置和电源管理接口（Advanced Configuration and Power Interface）](http://en.wikipedia.org/wiki/ACPI)构建出描述计算机中的设备的数据表，这些数据表之后会被内核使用。

加电自检过后 BIOS 需要启动操作系统了，它必须从某种介质中搜寻：硬盘，CD-ROM，软盘等等。BIOS 寻找的顺序可以由用户配置。如果没有可启动的设备，BIOS 会停止并显式错误，例如 "Non-Systme Disk or Disk Error"。这种症状通常是硬盘损坏了。BIOS 找到一个可启动的硬盘后会从硬盘开始启动。

接着 BIOS 读取硬盘的前 512 字节（0 [扇区](http://en.wikipedia.org/wiki/Disk_sector)）。这个扇区被称作[主引导记录（Master Boot Record）](http://en.wikipedia.org/wiki/Master_boot_record)，通常包含两个重要的组件：一个微小的引导程序和一个硬盘的分区表。BIOS 将 MBR 中的内存载入内存的 0x7c00 地址，并跳转到此位置开始执行 MBR 中的代码。

![](/img/in-post/how-computers-boot-up/mbr.png)

MBR 中的代码可以是一个 Windows MBR Loader，也可以是 Linux Loader 例如 LILO 或 GRUB，甚至可能是一个病毒。分区表是固定的，一个 64 字节的区域，每 16 字节为一个条目，描述了硬盘的分区情况。通常，微软的 MBR 程序在分区表中寻找唯一的活动分区，加载此分区的引导扇区，并执行其中的代码。**引导扇区**是分区的第一个扇区。如果分区表有错误，你可能会看到 "Invalid Partitoin Table" 或 "Missing Operating System" 之类的错误信息。这些信息并不是由 BIOS 产生的，而是由 MBR 中的代码产生的。因此具体的错误信息取决于 MBR 中代码的版本。

随着计算机发展，引导过程变得越来越复杂而灵活。Linux 的引导程序 Lilo 和 GRUB 可以处理各种不容的操作系统，文件系统和引导配置。它们的 MBR 指令并不是「从活动分区启动」。整个过程会类似下面这样：

1. MBR 中包含引导程序的第一阶段。GRUB 中被称作 stage 1.

2. 因为 MBR 空间太小了，其中的代码仅能加载另硬盘中包含了额外引导代码的其他扇区。这个扇区可能是一个分区的引导扇区，也可能是在 MBR 安装时硬编码到 MBR 代码中的某个扇区。

3. MBR 中的代码和上一步中加载的代码会读取包含了引导扇区第二阶段代码的文件。在 GRUB 中被称作 stage 2，在 Windows Server 中它是 c:\NTLDR. 如果第二步失败了，在 Windows 你可能会看到类似 "NTLDR is missing" 的错误信息。stage 2 的代码随后会读取配置文件（例如，GRUB 中是 grub.conf，Windows 中是 boot.ini）。然后它会显示启动选项，或者直接启动系统。

4. 这时候引导程序需要启动内核了。它必须能够识别内核所在的引导分区的文件系统。在 Linux 中，这一步是读取一个类似 "vmlinuz-2.6.22-14-server" 的文件，加载至内存中并跳转至内核启动代码。在 Windows Server 2003 中，内核的启动代码和内核镜像本身是分开的，启动代码嵌入到了 NTLDR 中。在进行一些初始化后，NTLDR 从 c:\Windows\System32\ntoskrnl.exe 载入内核镜像，而 Linux 中，GRUB 只需要跳转至内核的入口点。

这里有个复杂的情况需要注意（我告诉过你这个过程使用了很多黑科技）。Linux 内核的镜像文件，即使是压缩过的，也无法载入真实模式下的 640 K 可用内存。引导程序需要调用 BIOS 程序来读取硬盘，因此必须运行在真实模式下，然后这时候内存空间明显不够用来装载内核。这里是解决办法是[非真实模式（unreadl mode）](http://en.wikipedia.org/wiki/Unreal_mode)。这实际上不是一个 CPU 模式，而是一个在真实模式和保护模式之间切换的技术，使得 CPU 可以寻址超过 1MB 的内存空间，并且仍然在使用 BIOS。如果你去读 GRUB 的源代码，你会到处都是这样的转换（在 statge2/ 目录下的 real_to_prot 和 prot_to_read 调用）。无论如何，这之后 CPU 将内核完全读入内存，并且读入之后 CPU 仍处于真实模式。

我们再回过头来看第一张图，从引导程序到早起的内核初始化过程，这时内核解压开始工作了。下篇文章会介绍内核初始化的过程。
