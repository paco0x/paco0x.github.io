---
layout:     post
title:      "Linux 共享库指南"
subtitle:   "Shared Library in Linux"
date:       2014-11-20 21:00:00
author:     Liao
catalog:    true
header-img: img/post-bg-2015.jpg
permalink:  /linux-shared-libary/
tags:
    - Operating System
---

初学 Linux 时，常常被系统中各种库文件和链接弄得摸不着头脑，它们分别有什么作用？本文解释了 Linux 中使用共享库时的约定与行为。


## 动态库和静态库

库文件是编译好的程序，这类程序通常没有**执行入口**，需要其他程序的调用才会执行。

Linux 中的 so（shared object，共享对象） 也叫做动态链接库，在 Linux 中是一系列以 .so 为后缀名（后面可能还跟有版本号）的文件。与动态链接库所对应的是静态库，静态库文件是以 .a 为后缀名的文件。

**使用动态链接库的程序**

- 程序在启动时操作系统会将所需的动态链接库加载到内存中供程序使用，搜索的路径是系统默认或者自定义的

- 使用动态链接库的程序会比较小，但是使用时对操作系统环境有一定要求

- 动态连接库在内存中是共享的，多个进程可以使用操作系统内存中的同一个动态链接库，因此一定程度上能够节省内存开销

- 库文件的更新不需要更新整个程序

**使用静态库的程序**

- 在编译时就将静态库编译到程序内部，对操作系统环境要求较低

- 静态库在内存中不能够共享，不能节省内存开销

- 将静态库编译进程序，程序的体积可能较大

- 库文件的修改需要重新编译整个程序

## Linux系统共享对象 (so) 的使用约定

在 Linux 系统中，只要动态链接库的安装得当，Linux 使用了一套机制，使得系统中的其他程能够自动的使用到最新的动态库文件。

### 共享对象名

在Linux系统中，共享对象一共有3个名字，分别是 "soname"，"real name" 和 "linker name"

#### soname

每一个共享库文件都有一个特殊的名称叫做 soname，soname 包含 "lib" 前缀，库名称，".so" 后缀，最后跟上更新时会增长的版本号。一个完全限定 soname ( *Fully-qualified soname* ) 包含它所在目录的前缀，如 `/lib64/libgmodule-2.0.so.0`。完全限定 soname 的库文件通常是一个指向真正库文件名( real name )的**软链接**。

#### real name

real name 即包含库文件代码的真实文件名。库文件的名字通常是 soname 加上库文件的次版本号，发布版本号等，发布版本号可以省略。

如 `libgmodule-2.0.so.0.1200.3`。次版本号和发布版本号能够标明该库文件的具体版本。

#### linker name

linker name 是编译器链接器使用库文件时所使用的名字，linker name 是 soname 去掉版本号，如 `libgmodule-2.0.so`。linker name 通常是指向库文件 real name 或 soname 的软链接文件。

#### 各名字的使用
Linux 系统管理共享对象的关键点就是区分开这些名字。

程序在内部使用库文件时，应该使用 soname，这样即使库文件升级，真实文件名( real name )改变，soname 仍然可以保持名称不变，指需要重新指向新文件即可。

当创建一个库文件时，是需要创建 real name 的库文件（包含具体的版本信息）。当安装新版本的库文件时，将它们安装在系统默认的几个文件夹中(`/lib`，`/lib64`，`/usr/lib`，`/usr/lib64`)，执行 ldconfig 命令，ldconfig 会在系统默认的库文件目录和指定目录( `/etc/ld.so.conf` 和 `/etc/ld.so.conf.d/*.conf` )中检索所有的文件，并自动对 real name 的真实库文件创建 soname 的软链接，并将文件路径缓存至 `/etc/ld.so.cache` 文件中。

ldconfig 不会设置库文件的 linker name，这通常是库文件安装时设置的，linker name 通常是指向对应最新版本库文件 soname 的软链接文件。不自动设置 linker name 的原因是开发时，可能会需要使用旧版本的库文件。

举例来说，对于 libreadline 这个库，`/usr/lib64/libreadline.so.5` 是其完全限定 soname，ldconfig 命令会将其链接至 real name 库文件如 `/usr/lib64/libreadline.so.5.1`。还应该有一个 linker name 供编译器使用，如 `/usr/lib64/libreadline.so`，它是 `/usr/lib64/libreadline.so.5` 的软链接。

### 共享库文件的存放

对于共享库文件的存放，大多数开源软件遵循 GNU 标准。GNU 标准建议软件安装时将库文件安装至 `/usr/local/lib` 目录，将可执行文件安装至 `/usr/local/bin`目录。

而 FHS( Filesystem Hierarchy Standard ) 标准则建议将库文件安装至 `/usr/lib`，开机所需使用的库文件则安装至 `/lib` 目录（防止 `/usr` 目录单独分区挂载）, 其他非操作系统的库文件安装至 `/usr/local/lib`。

因此使用源代码安装的软件，库文件会默认安装至 `/usr/local/lib`，而使用包管理软件或系统安装时所安装的软件，库文件被安装至 `/lib` 和 `/usr/lib` 中。

如果库文件是被其他库文件调用的，则安装至 `/usr/local/libexec` (在发行版中通常是 `/usr/libexec`)。
 
X-windows 使用的库文件安装至 `/usr/X11R6/lib`。

`/lib/security` 中存放的是 PAM 模块，这些模块通常也是作为动态库被加载的。

## 库文件是怎样被使用的

在基于 GNU glibc 的系统中，包括所有的 Linux 操作系统，启动一个可执行的带链接库程序时，会自动启动一个**载入程序**。 在Linux系统中，这个载入程序叫做 `/lib/ld-linux.so.X`( X是版本号 )，64 位系统中这个文件是 `/lib64/ld-linux-x86-64.so.X`。这个载入程序会查找并载入程序所使用的其他共享库至内存中。

共享库文件查询的目录存放在 `/etc/ld.so.conf` 中。注意，对于红帽 Linux 系统，`/etc/ld.so.conf` 这个文件中并没有包含 `/usr/local/lib` 这个常用的库文件目录。

如果只是想要覆盖一个库文件的某些函数，但保留其余的内容，可以将覆盖库文件名（.o 后缀文件）保存至 `/etc/ld.so.preload` 文件中。这些覆盖库文件会比标准库文件优先读取，这通常用于紧急的版本补丁。

每次都查找 `/etc/ld.so.conf` 中的目录是很低效的，因此 ldconfig 程序会读取 `/etc/ld.so.conf` 文件中的所有目录中的文件，将库文件 real name 设置合适的软链接 soname，并将这些文件名缓存至 `/etc/ld.so.cache` 中。这样会大大加快库文件的寻找速度。

特殊的库文件 `linux-vdso.so`，在早期的 x86 处理器中，用户程序与管理服务之间的通信通过软中断实现。 随着处理器速度的提高，这已成为一个严重的瓶颈。 自 Pentium® II 处理器开始，Intel® 引入了 *Fast System Call* 装置来提高系统调用速度， 即采用 SYSENTER 和 SYSEXIT 指令，而不是中断。

`linux-vdso.so.1` 是个虚拟库，即 *Virtual Dynamic Shared Object*，它只存在于程序的地址空间当中。 在旧版本系统中该库为 `linux-gate.so.1`。 该虚拟库为用户程序以处理器可支持的最快的方式 （对于特定处理器，采用中断方式；对于大多数最新的处理器，采用快速系统调用方式） 访问系统函数提供了必要的逻辑 。

## 共享库相关的环境变量

**LD_LIBRARY_PATH**

在 Linux 系统中，LD_LIBRARY_PATH 是冒号分隔的一组目录，这些目录在查找库文件时将会比标准的库文件目录优先查找。在 debug 一个新共享库或者在特殊目的中使用非标准的库文件时这个变量会很有用。LD_LIBRARY_PATH 对于开发和测试很有用，但最好不要作为生产环境使用。

如果不想使用 LD_LIBRARY_PATH 变量，可以直接调用载入程序并传递参数，例如：

`/lib/ld-linux.so.2 --library-path PATH EXECUTABLE`

**注意：在生产环境中一定不要使用 LD_LIBRARY_PATH**，详情可以参考 [LD_LIBRARY_PATH Is Not The Answer](http://prefetch.net/articles/linkers.badldlibrary.html)，[Why LD_LIBRARY_PATH is bad](http://xahlee.info/UnixResource_dir/_/ldpath.html)，[Avoiding LD_LIBRARY_PATH](https://blogs.oracle.com/ali/entry/avoiding_ld_library_path_the)

这里总结几个原因：

- LD_LIBRARY_PATH 中的库文件会比系统默认库文件载入的优先级更高，如果在 LD_LIBRARY_PATH 中的目录权限设置不当，导致他人放入了带有后门的版本的系统核心库，系统的安全性将大大降低，这是为什么具有 suid 属性的程序会完全的忽略掉 LD_LIBRARY_PATH 这个变量。

- 由于 LD_LIBRARY_PATH 定义的目录中的库文件将被优先加载，因此当系统更新了库文件位于 `/lib64/` 等目录中后，系统还是会使用老版本的库文件。

- 有时一些预编译的程序可能使用特定位置的第 3 方动态库文件，理想的情况是将这些库文件一同拷贝至主机中，并在预安装过程中重新链接库文件。另一种做法是在一个包装程序或脚本中定义 LD_LIBRARY_PATH 变量指定库文件，这种情况下，LD_LIBRARY_PATH 是会被继承至程序中的，从而可能扰乱系统运行。

**LD_PRELOAD**

作用和 `/etc/ld.so.preload` 一样


参考：

- [Program Library HOWTO](http://tldp.org/HOWTO/Program-Library-HOWTO/shared-libraries.html)
- [学习 Linux，101: 管理共享库](http://www.ibm.com/developerworks/cn/linux/l-lpic1-v3-102-3/)
