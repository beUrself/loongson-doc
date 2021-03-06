概述
====

龙芯 3B1500 八核处理器采用 32nm 工艺制造，在单个芯片内集成了 8 个 64 位超标量通
用向量处理器核，最高工作主频为 1.2GHz，主要特征如下：

 - 片内集成 8 个 64 位的四发射超标量 GS464v 高性能向量处理器核；
 - 片内集成 8 核共享的 8MB 三级 Cache；
 - 片内集成 2 个 72 位 667MHz 的 DDR2/3 控制器（64 位总线+8 位 ECC）
 - 片内集成 2 个 16 位 1600MHz 的 HyperTransport 控制器；每个 16 位的 HT 端口可
   以拆分成两个 8 路的 HT 端口使用； - 片内集成 2 个 UART、1 个 SPI、16 路 GPIO
   接口；
 - 片内集成 1 个非标准电压（1.8v）LPC 接口；
 - 片内集成 1 个非标准电压（1.8v）PCI 接口；
 - 支持多核芯片通过 HyperTransport 接口互连和跨芯片的全局 Cache 一致性；
 - 采用 FC-BGA-1121 封装。

\noindent 龙芯 3B1500 的芯片整体架构基于两级互连实现，芯片结构和介绍详见《龙芯
3B1500 用户手册（上）》关于龙芯 3B1500 简介。
