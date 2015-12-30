\chapter{I/O 中断}

龙芯 3A 芯片支持 32 个中断源，以统一方式管理。图~\ref{fig:interrupt}
给出了这些中断对应的中断源，以及中断的路由的简单示意。

\begin{figure}[htbp]
  \centering
  \setlength\fboxsep{15pt}
  \setlength\fboxrule{.5pt}
  %\fbox{\includegraphics[width=.7\linewidth,height=12cm]{gs3a-introute}}
  \caption{龙芯 3A 处理器中断路由示意图}
  \label{fig:interrupt}
\end{figure}

表~\ref{tab:intreg} 列出了这 32 个中断的配置寄存器。这些中断相关配置寄存器
都是以位的形式对相应的中断线进行控制的：任意一个 IO
中断源可以被配置为是否使能、触发的方式、以及被路由的目标处理器核中断脚。
\begin{table}[h]
  \centering
  \begin{tabular}{|l|c|c|l|} \hline
    \cellalign{|c|}{名称} & 地址 & 位宽 & \cellalign{|c|}{描述} \\ \hhline
    Entry0-31     & 0x3FF0\_1400-0x3FF0\_141F  & 8    & 32 个 8 位中断路由寄存器 \\
    Intisr        & 0x3FF0\_1420   & 32   & 中断状态寄存器 \\
    Inten         & 0x3FF0\_1424   & 32   & 中断使能状态寄存器 \\
    Intenset      & 0x3FF0\_1428   & 32   & 设置使能寄存器 \\
    Intenclr      & 0x3FF0\_142c   & 32   & 清除使能寄存器 \\
    Intedge       & 0x3FF0\_1438   & 32   & 触发方式寄存器 \\
    Core0\_Intisr & 0x3FF0\_1440   & 32   & 路由给 Core0 的中断状态寄存器 \\
    Core1\_Intisr & 0x3FF0\_1448   & 32   & 路由给 Core1 的中断状态寄存器 \\
    Core2\_Intisr & 0x3FF0\_1450   & 32   & 路由给 Core2 的中断状态寄存器 \\
    Core3\_Intisr & 0x3FF0\_1458   & 32   & 路由给 Core3 的中断状态寄存器 \\ \hline
  \end{tabular}
  \caption{I/O 中断配置寄存器}
  \label{tab:intreg}
\end{table}

使能 （Enable） 配置由三个寄存器控制： Inten、Intenset、和 Intenclr：
每个中断与中断源的对应关系，和使能寄存器的属性配置见表~\ref{tab:intenconf}。
\begin{itemize}
  \item Inten 寄存器读取当前各中断使能的情况；
  \item Intenset 设置中断使能：Intenset 寄存器写 1 的位对应的中断被使能；
  \item Intenclr 清除中断使能：Intenclr 寄存器写 1 的位对应的中断被清除；
  \item Intedge 配置寄存器选择中断信号是否采用脉冲形式的（如 PCI\_serr）；写 1
    表示脉冲触发，写 0 表示电平触发。中断处理程序可以通过设置 Intenclr
    的相应位来清除脉冲记录。
\end{itemize}

\begin{table}[h]
  \centering
  \begin{tabular}{|c|l|c|c|c|c|} \hline
          &                 & \multicolumn{4}{c|}{访问属性/缺省值} \\ \cline{3-6}
    位域  & \cellalign{c|}{中断源} & Intedge & Inten & Intenset & Intenclr \\ \hhline
    31:24 & HT1\_Int7-0     & RW/0    & R/0   & RW/0     & RW/0 \\
    23:16 & HT0\_Int7-0     & RW/0    & R/0   & RW/0     & RW/0 \\
    15    & PCI\_perr和serr & RW/0    & R/0   & RW/0     & RW/0 \\
    14    & 保留            & RW/0    & R/0   & RW/0     & RW/0 \\
    13    & Barrier         & RW/0    & R/0   & RW/0     & RW/0 \\
    12:11 & MC1-0           & RW/0    & 保留  & 保留     & 保留 \\
    10    & LPC             & R/1     & R/0   & RW/0     & RW/0 \\
    9     & MT\_Int1        & R/1     & R/0   & RW/0     & RW/0 \\
    8     & MT\_Int0        & R/0     & R/0   & RW/0     & RW/0 \\
    7:4   & PCI\_Int3-0     & R/0     & R/0   & RW/0     & RW/0 \\
    3:0   & SYS\_Int3-0     & RW/0    & R/0   & W/0      & W/0  \\ \hline
  \end{tabular}
  \caption{中断使能控制寄存器属性}
  \label{tab:intenconf}
\end{table}

这 32 个 I/O 中断源每个都对应一个 8 位的路由配置（Entry）寄存器。龙芯 3A
中集成了 4 个处理器核。 软件可以通过设置对应的路由配置寄存器，选择中断的
目标处理器核，以及路由到处理器核中断 Int0 到 Int3 中的任意一个（即对应
CP0\_Status 的 IP2 到 IP5 中的一个??）。
路由寄存器采用向量的方式进行路由选择，其格式如图~\ref{fig:intentry} 所示。
\setlength{\bitwidth}{1.2cm}
\begin{figure}[h]
  \begin{center}
    \begin{bytefield}{8}
      \bitheader[b]{7,4,3,0} \\
      \bitbox{4}{路由的处理器核中断号} &
      \bitbox{4}{路由的处理器核向量号}
    \end{bytefield}
  \end{center}
  \caption{中断路由寄存器位域图}
  \label{fig:intentry}
\end{figure}
例如 Entry2=0x48 表明 2 号中断被路由到 3 号处理器的 INT2 上。 (why??)

\noindent \remark{问题：}
\begin{itemize}
  \item 同一中断可以同时dispatch到几个核吗？如果有冲突，硬件如何处理？
  \item MT\_Int0 和 MT\_Int1 的缺省值不同。有什么原因吗？
  \item I/O interrupt 和 IPI 中断的关系？
\end{itemize}

\section{处理器核间中断与通信}

龙芯 3A 为每个处理器核都实现了 8 个核间中断寄存器（inter-processor
interrupt，简称 IPI）以支持多核 BIOS
启动和操作系统运行时在处理器核之间进行中断和通信。龙芯 3A
的四个核间中断寄存器块的基地址分别为：
\begin{table}[h]
  \centering
  \begin{tabular}{|c|c|c|c|c|} \hline
               & Core0 & Core1 & Core2 & Core3 \\ \cline{2-5}
    IPI 基地址 & 0x3FF0\_1000 & 0x3FF0\_1100 & 0x3FF0\_1200 & 0x3FF0\_1300 \\ \hline
  \end{tabular}
  \caption{核间中断寄存器块的基地址}
  \label{tab:ipibases}
\end{table}

每个核间中断寄存器块 8 个寄存器的说明见表~\ref{tab:ipireg}。其中四个缓存寄存器
(\remark{这个名字好像不太好，就用邮箱寄存器如何？})
IPI\_Mailbox0-3 用于供启动时传递参数使用，按 64 或者 32
位的非缓存方式进行访问。

\begin{table}[h]
  \centering
  \begin{tabular}{|l|c|c|c|p{9cm}|} \hline
    名称          & 偏移 & 权限 & 位宽 & \cellalign{c|}{描述} \\ \hhline
    IPI\_Status   & 0x00 & R    & 32 & 状态寄存器: 任何一位被置 1，且对应位使能情况下，处理器核 INT4 (??) 中断线被置位 \\
    IPI\_Enable   & 0x04 & RW   & 32 & 使能寄存器: 控制对应中断位是否有效 \\
    IPI\_Set      & 0x08 & W    & 32 & 置位寄存器: 在任何位写 1，对应的状态寄存器位被置 1 \\
    IPI\_Clear    & 0x0C & W    & 32 & 清除寄存器: 在任何位写 1，对应的状态寄存器位被清 0 \\
    IPI\_MailBox0 & 0x20 & RW   & 64 & 缓存寄存器 0 \\
    IPI\_MailBox1 & 0x28 & RW   & 64 & 缓存寄存器 1 \\
    IPI\_MailBox2 & 0x30 & RW   & 64 & 缓存寄存器 2 \\
    IPI\_MailBox3 & 0x38 & RW   & 64 & 缓存寄存器 3 \\ \hline
  \end{tabular}
  \caption{处理器核间中断寄存器}
  \label{tab:ipireg}
\end{table}

表~\ref{tab:ipireg} 列出的是单个龙芯 3A
芯片所组成的单节点多处理器系统的的核间中断相关寄存器列表。在采用多片龙芯 3A
互连构成多节点 CC-NUMA
系统时，每个芯片节点对应一个系统全局节点编号，节点内处理器核的 IPI
寄存器地址与其所在节点的基地址成固定偏移关系。例如，1 号节点
(其节点基地址为0x1000\_0000\_0000) 的 2 号处理器 IPI\_Enable 地址为
\begin{center}
  Core2\_IPI\_Enable: 0x1000\_0000\_0000 + 0x3FF0\_1200 + 0x04 = 0x1000\_3FF0\_1204。
\end{center}
其他节点，其他处理器核，其他 IPI 寄存器依次类推。
