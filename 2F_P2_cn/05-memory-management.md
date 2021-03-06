内存管理
========

龙芯 GS464 处理器核提供了一个功能完备的内存管理单元（MMU），它利用片上的 TLB 实
现虚拟地址到物理地址的转换。

本章节描述了处理器核的虚拟地址和物理地址空间，虚拟地址到物理地址的转换，TLB 在实
现这 些转换时的操作，Cache，以及为 TLB 提供软件接口的系统控制协处理器（CP0）寄存
器。


快速查找表 TLB
--------------

把虚拟地址映射成物理地址通常是由 TLB 来实现的[^note:nonmap]。第一级的 TLB 是
JTLB，同时也作为数据 TLB ，另外，龙芯 GS464 处理器核包含独立的指令 TLB 以缓解对
JTLB 的竞争。

[^note:nonmap]: 也有不经过 TLB 的虚拟地址转换，比如 CKSEG0 和 CKSEG1 内核地址空
间段（见图\ \ref{fig:addr-kernelmode}）就是不进行页面映射的，其中的物理地址是由
虚拟 地址减去一个基址得到的。

JTLB
----

为了能够快速地进行虚拟地址到物理地址的映射，龙芯 GS464 处理器核采用了较大的，全
相联 映射机制的 TLB，JTLB 用于指令和数据的地址映射，用它们的进程号和虚拟地址进行
索引。

JTLB 按奇/偶表项成对组织，把虚拟地址空间和地址空间标识符映射到 256T 的物理地址空
间。 在默认的情况下，JTLB 有 64 对奇/偶表项，允许 128 页进行映射。有两个机制分别
用来协助控制映射空间的大小和内存不同区域的替换策略。

第一，页的大小可以是 4KB 到 16MB，但必须是按 4 倍递增。CP0 寄存器 PageMask 用于
记录映 射的页的大小，并且这个记录在写一个新的表项的同时载入 TLB 中。龙芯 GS464
处理器核可以在同 一运行时刻支持不同大小的页，允许操作系统产生特定目的的映射：例
如，视频编解码处理中帧缓冲 区就可以只用一个表项来进行内存映射。

第二，龙芯 GS464 处理器核在 TLB 缺失的时候可以采用随机替换的策略来选择要被替换的
TLB 表项。操作系统也可以把一定数量的页面驻留在 TLB 中，而不致于被随机替换出去，
这种机制有利 于使操作系统提高性能，避免死锁。这种机制也使实时系统比较方便地为某
一关键软件提供特定入口。

此外对每个页来说，JTLB 还维护该页面的 Cache 一致性属性，每个页都有特定的位来标记
：不 经过 Cache（Uncached），非一致性 Cache（Cacheable Noncoherent），或者是非
Cache 加速（Uncached Accelerated）。


### 指令 TLB

龙芯 GS464 处理器核的指令 TLB（ITLB）有 16 个表项，它最小化了 JTLB 的容量，并通
过一个 大的相联阵列缩短了映射时的时间关键路径，降低了功率。每个 ITLB 表项只能映
射一页，页面大小 由 PageMask 寄存器来指定。ITLB 指令地址的映射和数据地址的映射能
并行执行，从而提高了性能。 当 ITLB 中的表项失效时，从 JTLB 中查找相应的表项，随
机选择一个 ITLB 表项进行替换，ITLB 的 操作对用户是完全透明的。处理器保证 ITLB 与
JTLB 的一致，当使用核心态指令修改 JTLB 时，ITLB 将被自动清空。

### 命中和失效

如果虚拟地址与 TLB 中某个表项的虚拟地址一致（即 TLB 命中），则物理页面号就从 TLB
中取 出，并和偏移连接组成物理地址。

如果虚拟地址与 TLB 中任何表项的虚拟地址都不一致（即 TLB 失效），则 CPU 产生一个
异常， 并由软件根据内存中存放的页表重新填写 TLB。软件既可以重写指定的 TLB 表项，
也可以使用硬件 提供的机制重写任意一个 TLB 表项。

### 多项命中

龙芯 GS464 处理器核对 TLB 中虚地址不只与一个表项的虚地址一致的情况没有提供任何的
探测 和禁用机制，这一点不像早期的 MIPS 处理器的设计。多项命中并不会物理地破坏
TLB，因此多项命 中的探测机制是不必要的。但是多项命中的情况并没有被定义，因此软件
要控制不要让多项命中的情 况发生。


处理器模式
----------

龙芯 GS464 处理器核有 3 种工作模式，但是与其它 MIPS 处理器不同，龙芯 GS464 处理
器核只支持一种地址模式，一种指令集模式和一种尾端模式。


### 处理器工作模式

以下三种模式的处理器优先级依次降低：

* 内核模式（最高的系统优先级）：在这种模式下处理器可以访问和改变任何寄存器，操作
  系统最 内层的内核运行在内核模式；
* 管理模式：处理器的优先级降低，操作系统的一些不太关键的部分运行在该模式；
* 用户模式（最低的系统优先级）：该模式下使不同的用户间不致互相干扰。

三种模式的切换是由操作系统（在内核模式）置位状态寄存器 KSU 域的相应位来实现的。
当出 现一个错误（ERL 位置位）或出现一个例外（EXL 位置位）时，处理器被强制切换到
内核模式。表\ \ref{tab:cpu-modes} 列出了三种模式切换时 KSU、EXL 和 ERL 的置位情
况，空的表项可以不必关心。

Table: 处理器的工作模式 \label{tab:cpu-modes}

|  KSU  |  ERL  |  EXL   |   描述    |
| ----- | ----- | ------ | --------- |
| 4:3   | 2     | 1      |           |
| 10    | 0     | 0      | 用户模式  |
| 01    | 0     | 0      | 管理模式  |
| 00    | 0     | 0      | 内核模式  |
|       | 0     | 1      | 例外级别  |
|       | 1     |        | 错误级别  |

### 地址模式

龙芯 GS464 处理器核只支持 64 位的虚拟地址模式，并且硬件保证兼容 32 位的地址模式。


### 指令集模式

龙芯 GS464 处理器核实现了完备的 MIPS64R2 指令集， 另外还增加了一些整型和浮点指令
，增 加的指令见附件 A 和附录 B。

### 尾端模式

龙芯 GS464 处理器核只工作在小尾端模式。

地址空间 \label{sec:addrspace}
--------

本节叙述的是虚拟地址空间，物理地址空间，和经过 TLB 进行虚实地址转换的方法。

### 虚拟地址空间

龙芯 GS464 处理器核有三个虚拟地址空间：用户地址空间、管理地址空间和内核地址空间
，每 个空间都是 64 位的，并且包含一些不连续的地址空间段，最大的段为 256T(248)字
节。 \ref{ssec:useraddr} 节到 \ref{ssec:kerneladdr} 节分别描述了这三种地址空间。

### 物理地址空间

通过使用 48 位地址，处理器的物理地址空间大小为 256T(248)字节。以下小节将详述虚实
地址转 换的方法。

### 虚实地址转换

进行虚实地址转换时，首先比较处理器给出的虚拟地址和 TLB 中存放的虚拟地址。当虚页
号 （VPN）等于某个 TLB 表项的 VPN 域，并且如果下面两种情况中的任何一种成立：

* TLB 表项的 Global 位为 1
* 两个虚拟地址的 ASID 域一样。

TLB 就命中了。如果不满足以上条件，那么 CPU 会产生 TLB 失效异常，以使软件能够根据
内存中存放的页表重新填写 TLB。如果 TLB 命中了，则物理页号将从 TLB 中取出，并与页
内偏移量 Offset 合并，形成物理地址。页内偏移量 Offset 在虚实地址转换的过程中不经
过 TLB。

图 \ref{fig:v2p} 所示为虚实地址转换，虚拟地址被一个 8 位的地址空间标识符（ASID）
扩展了，该措施降低了上下文切换时进行 TLB 刷新的频率。ASID 存放在 CP0 EntryHi 寄
存器中。Global 位（G）在 相应的 TLB 表项中。地址转换是，虚页号（VPN）表示的虚拟
地址（VA）与 TLB 中的对应域作比较； 如果有一致的情况，则物理地址（PA）高位的页框
号（PFN）从 TLB 中输 出；偏移量 Offset TLB PFN 合并形成物理地址。

\begin{figure}[htbp]
\centering
\includegraphics[scale=0.6]{../images/addr-virtual2physical.pdf}
\caption{虚实地址转换概览 \label{fig:v2p}}
\end{figure}

图 \ref{fig:addrtrans-64bit} 显示了 64 位模式的虚实地址转换过程，这个图显示了最
大页面 16MB 和最小页面 4KB 的 情况。 图的上半部分显示了页面大小为 4K 字节的情况
，页内偏移量 Offset 占用虚拟地址中的 12 位，虚 拟地址中剩下的 36 位为虚页号 VPN
，用于索引 64G 个页表表项； 图的下半部分显示了页面大小为 16M 字节的情况，页内偏
移量 Offset 占用虚拟地址中的 24 位， 虚拟地址中剩下的 24 位为虚页号 VPN，用于索
引 16M 个页表表项。

\begin{figure}[htbp]
\centering
\includegraphics[scale=0.6]{../images/addr-v2p-16k-16m.pdf}
\caption{64 位模式虚拟地址转换 \label{fig:addrtrans-64bit}}
\end{figure}

### 用户地址空间 \label{ssec:useraddr}

在用户模式下，只有一个称为用户段（User Segment）的单独、统一的虚拟地址空间，其大
小为 256T（248）字节，名字为 XUSEG。

图 \ref{fig:addr-usermode} 显示了用户虚拟地址空间，可以在用户模式、管理模式、内
核模式下访问。 用户段从地址 0 处开始，当前活动的用户进程驻留在该段（XUSEG）中。
在不同模式下，TLB 对
XUSEG 段的映射处理方式都一样，并控制是否可以访问 Cache。 当处理器的 Status 寄存
器的值同时满足三个条件：KSU=102、EXL=0、ERL=0 时，处理器工作在 用户模式下。

![用户模式虚拟地址空间 \label{fig:addr-usermode}](../images/addr-usermode-cn.pdf)

所有可用的用户模式下虚拟地址的第 63 位到第 48 位必须都为 0，访问任何一个第 63 位
到第 40 位不全为 0 的地址都将导致地址错误异常，在 XUSEG 地址段的 TLB 缺失使用
XTLB 重填向量。龙 芯 GS464 处理器核的 XTLB 重填向量与 32 位模式下 TLB 的重填向量
有相同的例外入口地址。

### 管理地址空间 \label{ssec:addr-supmode}

管理模式是为分层结构的操作系统设计的。在分层结构的操作系统中，真正的内核运行在内
核模 式下，操作系统的其余部分运行在管理模式下。管理地址空间提供了管理模式下程序
访问的代码和数 据空间。管理地址空间的 TLB 缺失由 XTLB 重填处理器来处理。

管理模式和内核模式都可访问管理地址空间。 当处理器的 Status 寄存器的值同时满足三
个条件：KSU=012、EXL=0、ERL=0 时，处理器工作在 管理模式下。
图\ \ref{fig:addr-supmode} 显示了管理模式 下的用户和管理地址空间概况。

![管理模式虚拟地址空间 \label{fig:addr-supmode}](../images/addr-supmode-cn.pdf)

1. 64 位管理模式，用户地址空间（XSUSEG） \\
   在管理模式下，当访问用户地址空间并且 64 位地址的最高两位（第 63 和第 62 位）
   为 002 时，程序使用一个名字为 XSUSEG 的虚拟地址空间，XSUSEG 覆盖了当前用户地
   址空间 的全部 248（1T）字节。此时虚拟地址被扩展，加上 8 位的 ASID 域，形成一
   个系统中唯 一的虚拟地址。此地址空间从 0x0000 0000 0000 0000 开始，到 0x0000
   FFFF FFFF FFFF 结束。
1. 64 位管理模式，当前管理地址空间（XSSEG） \\
   在管理模式下，当 64 位地址的最高两位（第 63 和第 62 位）为 012 时，程序使用一
   个 名字为 XSSEG 的当前管理虚拟地址空间。此时虚拟地址被扩展，加上 8 位的 ASID
   域，形 成一个系统中唯一的虚拟地址。此地址空间从 0x4000 0000 0000 0000 开始，
   到 0x4000 FFFF FFFF FFFF 结束。
1. 64 位管理模式，独立管理地址空间（CSSEG）\\
   在管理模式下，当 64 位地址的最高两位（第 63 和第 62 位）为 112 时，程序使用一
   个 名字为 CSSEG 的独立管理虚拟地址空间。在 CSSEG 中的寻址与 32 位模式下在
   SSEG 中的 寻址是兼容的。此时虚拟地址被扩展，加上 8 位的 ASID 域，形成一个系统
   中唯一的虚拟 地址。此地址空间从 0xFFFF FFFF C000 0000 开始，到 0xFFFF FFFF
   DFFF FFFF 结束。

### 内核模式地址空间 \label{ssec:kerneladdr}

当处理器的 Status 寄存器的值满足下述条件：KSU=002 或 EXL=1 或 ERL=1 时，处理器工
作在内 核模式下。 每当处理器检测到一个例外时便进入内核模式，并一直保持到执行例外
返回指令（ERET）。 ERET 指令将处理器恢复到例外发生前所在的模式。

根据虚拟地址高位的不同，内核模式虚拟地址空间被分为不同的区域，如
图\ \ref{fig:addr-kernelmode} 所示。

![内核模式虚拟地址空间 \label{fig:addr-kernelmode}](../images/addr-kernelmode-cn.pdf)

1. 64 位内核模式，用户地址空间（XKUSEG）\\ 在内核模式下，当访问用户空间并且 64
   位虚拟地址的最高两位为 002 时，程序使用一 个 名字为 XKUSEG 的虚拟地址空间，
   XKUSEG 覆盖了当前用户地址空间。此时虚拟地址 被扩展 ，加上 8 位的 ASID 域，形
   成一个系统中唯一的虚拟地址。
1. 64 位内核模式，当前管理地址空间（XKSSEG） \\ 在内核模式下，当访问管理空间并且
   64 位地址的最高两位为 012 时，程序使用一个名 字 为 XKSSEG 的虚拟地址空间，
   XKSSEG 是当前管理虚拟地址空间。此时虚拟地址被扩 展，加 上 8 位的 ASID 域，形
   成一个系统中唯一的虚拟地址。
1. 64 位内核模式，物理地址空间（XKPHY）\\ 在内核模式下，当 64 位地址的最高两位为
   $10_2$ 时，程序使用一个名字为 XKPHY 的虚 拟地 址空间， XKPHY 是八个 248 字节
   的内核物理地址空间的集合。访问任何地址第 58 到第 48 位不为 0 的存储单 元都将
   引起地址错误。对 XKPHY 的访问不经过 TLB 进 行地址变换 ，而是将虚拟地址的第 47
   到第 0 位作为物理地址。虚拟地址的第 61 到 第 59 位控制是 否通过 Cache 和
   Cache 的一致性属性，与表\ ref{fig:tlbfig} 描述的 TLB 页的 C 位值含义相同。
1. 64 位内核模式，内核地址空间（XKSEG）
   * 在内核模式下，当 64 位地址的最高两位为 112 时，程序使用以下两个地址空间之一
     ： 内核虚拟地址空间 XKSEG，此时虚拟地址被扩展，加上 8 位的 ASID 域，形成一
     个系 统中唯一的虚拟地址；
   * 四个 32 位内核兼容地址空间，下一小节详述。
1. 64 位内核模式，兼容地址空间（CKSEG1：0，CKSSEG，CKSEG3）\\ 在内核模式下，64
   位地址的最高两位为 112，并且虚拟地址的第 61 到第 31 位所有位 都等于 1 时， 程
   序使用的以下四个 512M 字节地址空间中的一个，具体哪一个根据第 30、29 位决定
1. 内核模式下的用户、管理、内核地址空间概况
   1. CKSEG0：该 64 位虚拟地址空间不经过 TLB，与 32 位模式下的 KSEG0 兼容。
   Config 寄存 器的 K0 域控制是否通过 Cache 和 Cache 的一致性属性，
   1. CKSEG1：该 64 位虚拟地址空间不经过 TLB 也不经过 Cache，与 32 位模式下的
   KSEG1 兼 容。
   1. CKSSEG：该 64 位虚拟地址空间为当前管理虚拟地址空间，与 32 位模式下的 KSSEG
   兼 容 。
   1. CKSEG3：该 64 位虚拟地址空间为内核虚拟地址空间，与 32 位模式下的 KSEG3 兼
   容。

系统控制协处理器
----------------

系统控制协处理器(CP0)负责支持存储管理，虚实地址转换，例外处理，以及一些特权操作
。龙芯 GS464 处理器核有 26 个 CP0 寄存器和一个 64 项的 TLB，每个寄存器都有唯一的
寄存器号。下面的章节将给出与内存管理相关的寄存器的概述。

### TLB 表项的格式 \label{subsec:tlb-format}

表 \ref{fig:tlbfig} 表示 TLB 表项的格式，及表项中的每个域在 EntryHi，EntryLo0，
EntryLo1，PageMask 寄存器中的相应域。

\begin{table}[htbp]
\centering
\caption{TLB 表项的格式 \label{fig:tlbfig}}
\includegraphics[scale=0.85]{../images/tlbfig.pdf}
\end{table}

EntryHi，EntryLo0，EntryLo1，以及 PageMask 寄存器和 TLB 项的格式类似。唯一的不同
就是 TLB 项有一个 Global 域（G 位）， EntryHi 寄存器中没有，而是作为保留域出现。
表\ \ref{tab:cp0-entrylo}, \ref{tab:cp0-entryhi}, \ref{tab:cp0-pagemask}, 分别表
示了在图\ \ref{fig:tlbfig} TLB 项的各个对应域。

TLB 页一致性属性位（C）指定访问该页时是否需要通过 Cache，如果通过 Cache，则需要
选择 Cache 的一致性属性。表\ \ref{tab:tlb-cvalues} 表示 C 位对应的 Cache 一致性
属性。

Table: TLB 页的 C 位的值 \label{tab:tlb-cvalues}

| C(5:3) 值 | Cache一致性属性                           |
|-----------|-------------------------------------------|
| 0         | 保留                                      |
| 1         | 保留                                      |
| 2         | 非高速缓存（Uncached）                    |
| 3         | 非一致性高速缓存（Cacheable Noncoherent） |
| 4         | 保留                                      |
| 5         | 保留                                      |
| 6         | 保留                                      |
| 7         | 非高速缓存加速（Uncached Accelerated）    |


### CP0 寄存器

表 \ref{tab:registers-memory} 列出了与内存管理相关的 CP0 寄存器，第 3 章对 CP0
寄存器进行了完备的描述。

Table: 内存管理相关的 CP0 寄存器 \label{tab:registers-memory}

| 寄存器号 | 寄存器名 |
|----------|----------|
| 0        | Index    |
| 1        | Random   |
| 2        | EntryLo0 |
| 3        | EntryLo1 |
| 5        | PageMask |
| 6        | Wired    |
| 10       | EntryHi  |
| 15       | PRID     |
| 16       | Config   |
| 17       | LLAddr   |
| 28       | TagLo    |
| 29       | TagHi    |


### 虚拟地址到物理地址的转换过程

在虚地址到物理地址转换时，CPU 将虚地址的 8 位 ASID（如果全局位 G 没有设置）和
TLB 项的 ASID 进行比较，看是否匹配。在比较 ASID 的同时还需要根据页掩码（PageMask
）的值将虚地址的高 15~27 位和 TLB 项的虚页号进行匹配比较。如果有 TLB 项匹配，从
匹配的 TLB 项中取出物理地址和访问控制位（C，D 和 V）。对一个有效的地址转换来说，
匹配的 TLB 项的 V 位必须设置，但是在匹配比较时不考虑 V 位的值。
图\ \ref{fig:tlb-transaddr} 表示了 TLB 地址转换过程。

![TLB 地址转换 \label{fig:tlb-transaddr}](../images/tlb-transaddr2-cn.pdf)

### TLB 失效

如果没有任何一 TLB 项匹配虚地址，引发一个 TLB 不命中例外。如果访问控制位(D 和 V)
指示 访问不是合法的，引发一个 TLB 修改或者 TLB 无效例外。如果 C 位等于 0112，被
检索到的物理地址 通过 Cache 访问内存，否则不通过 Cache。

### TLB 指令

表 \ref{tab:mips64-tlb-ins} 列出了所有的 CPU 所提供的用于和 TLB 操作有关的指令。

\begin{inslongtable}{CP0 指令}{tab:mips64-tlb-ins}\hhline
  TLBR   & 读索引的 TLB 项     & MIPS32 \tabularnewline
  TLBWI  & 写索引的 TLB 项     & MIPS32 \tabularnewline
  TLBWR  & 写随机的 TLB 项     & MIPS32 \tabularnewline
  TLBP   & 在 TLB 中搜索匹配项 & MIPS32 \tabularnewline
\end{inslongtable}

### 代码例子

第一个例子是如何配置 TLB 表项来映射一对 4KB 的页面。实时系统的内核大多都这么做，
这种简单的内核 MMU 只用于进行内存保护，所以静态映射就足够了，在所有静态映射的系
统中所有 TLB 例外都被作为是错误条件（不可访问）。

   1.  mtc0 r0,C0_WIRED         -- make all entries available to random replacement
   2.  li r2, (vpn2<<13)|(asid & 0xff);
   3.  mtc0 r2, C0_ENHI         -- set the virtual address
   4.  li r2, (epfn<<6)|(coherency<<3)|(Dirty<<2)|Valid<<1|Global)
   5.  mtc0 r2, C0_ENLO0        -- set the physical address for the even page
   6.  li r2, (opfn<<6)|(coherency<<3)|(Dirty<<2)|Valid<<1|Global)
   7.  mtc0 r2, C0_ENLO1        -- set the physical address for the odd page
   8.  li r2, 0                 -- set the page size to 4KB
   9.  mtc0 r2,C0_PAGEMASK
   10. li r2, index_of_some_entry -- needed for tlbwi only
   11. mtc0 r2, C0_INDEX          -- needed for tlbwi only
       tlbwr                      -- or tlbwi

一个完备的虚拟存储操作系统（如 UNIX），用 MMU 进行内存保护，并进行主存和大容量存
储 设备的换页。这个机制使程序可以访问更大的存储设备而不仅仅局限于系统物理分配的
空间。这个依 赖于请求调页的机制需要动态页面映射。动态映射通过一系列不同类型的
MMU 例外实施，TLB 重填 是这种系统中最常见的例外。下面是一个可能的 TLB 重填例外控
制。

    12. refill_exception:
    13. mfc0 k0,C0_CONTEXT
    14. sra k0,k0,1           -- index into the page table
    15. lw k1,0(k0)           -- read page table
    16. lw k0,4(k0)
    17. sll k1,k1,6
    18. srl k1,k1,6
    19. mtc0 k1,C0_TLBLO0
    20. sll k0,k0,6
    21. srl k0,k0,6
    22. mtc0 k0,C0_TLBLO1
    23. tlbwr                 -- write a random entry
        eret

这个例外控制处理非常简单，因为它的经常执行会影响系统性能，这就是 TLB 重填例外分
配独 立的例外向量的原因。这段代码假设需要的映射在主存储器的页表中已经建立起来了
。如果没有建立 起来，那么在 ERET 指令后将发生 TLB 失效例外。TLB 失效例外很少发生
，这是有益的，因为它必 须计算期望的映射，并可能需要从后援存储器中读取部分页表。
TLB 修改例外用于实现只读页面和标 记进程清除代码需要修改的页。为了保护不同的进程
和用户不受相互的干扰，虚拟存储操作系统通常 在用户模式执行用户程序。下面的例子表
示如何从内核模式进入用户模式。

    24. mtc0 r10, C0_EPC                     -- assume r10 holds desired usermode address
    25. mfc0 r1, C0_SR                       -- get current value of Status register
    26. and r1,r1, ~(SR_KSU || SR_ERL)       -- clear KSU and ERL field
    27. or r1, r1, (KSU_USERMODE || SR_EXL)  -- set usermode and EXL bit
    28. mtc0 r1, C0_SR
        eret                                 -- jump to user mode

物理地址空间分布 \label{sec:40vs48}
----------------

龙芯 3 号的地址空间按照地址的高位均匀分布到各个结点。48 位地址的高 4 位[47:44]对
应地址空 间所在的结点编号，每个结点拥有固定的 44 位地址空间。而在结点内 44 位的
地址空间又进一步划分 为 8 个 41 位的地址空间，采用 41 位的空间主要是由于一个端口
可能会接两个 HT 控制器，而每个 HT 需要 40 位的地址空间。

