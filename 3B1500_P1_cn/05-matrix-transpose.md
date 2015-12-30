矩阵转置模块
============

龙芯 3A 中内置了两个独立于处理器核的矩阵转置模块，其基本功能是
通过软件的配置，实现对存放在内存中矩阵进行从源矩阵到目标矩阵的
行列转置功能。两个矩阵转置模块分别集成在龙芯 3A 的两个 HyperTransport
控制器内部(\remark{HT0还是HT1，还是一个控制器一个？})，通过 1 级交叉开关实现对二
级 Cache 及内存的读写。这两个转置模块的基地址为：
\begin{table}[h]
  \centering
  \begin{tabular}{|c|c|c|} \hline
           & 转置模块0 & 转置模块1 \\ \cline{2-3}\cline{2-3}
    基地址 & 0x3FF0\_0600 & 0x3FF0\_0700 \\ \hline
  \end{tabular}
  \caption{矩阵转置模块基地址}
  \label{tab:mtbase}
\end{table}

对每一个转置模块，由于转置前同一 Cache 行的元素顺序在转置后的矩阵中是分散的，为
了提高读写效率，需要读入多行数据，使得这些数据可以在转置后的矩阵中以 Cache 行为
单位进行写入，因此在矩阵转置模块中设置了一个大小为 32 行的缓存区，实现横向方式写
入（从源矩阵读入到缓冲区），纵向读出（由缓冲区写入到目标矩阵）。

矩阵转置的工作过程为先读入 32 行源矩阵数据，再将该 32 行数据写入到目标矩阵，依次
下去，直至完成整个矩阵的转置。矩阵转置模块还可以根据需要，仅进行预取源矩阵而不写
目标矩阵，以此方式来实现对数据进行预取到 2 级 Cache 的操作。转置涉及的源矩阵可能
是位于一个大矩阵中的一个小矩阵，因此，其矩阵地址可能不是完全连续，相邻行之间的地
址会有间隔，需要实现更多的编程控制接口。下面表 \ref{tab:mtconf} 和
\ref{tab:mttransctrl} 说明了每个矩阵转置涉及到的编程接口。

\begin{table}
  \centering
  \begin{tabular}{|c|l|c|l|} \hline
    地址偏移 & \cellalign{|c|}{名称} & 属性 & \cellalign{|c|}{说明} \\ \hhline
    0x00     & src\_start\_addr & RW   & 源矩阵起始地址 \\
    0x08     & dst\_start\_addr & RW   & 目标矩阵起始地址 \\
    0x10     & row              & RW   & 源矩阵的一行元素个数 \\
    0x18     & col              & RW   & 源矩阵的一列元素个数 \\
    0x20     & length           & RW   & 源矩阵所在大矩阵的行跨度（字节） \\
    0x28     & width            & RW   & 目标矩阵所在大矩阵的行跨度（字节） \\
    0x30     & trans\_ctrl      & RW   & 转置控制寄存器 (具体位域解释见表\ref{tab:mttransctrl}) \\
    0x38     & trans\_status    & RO   & 转置状态寄存器 ([0]: 源矩阵读完毕, [1]: 目标矩阵写完毕) \\ \hline
  \end{tabular}
  \caption{矩阵转置编程接口说明}
  \label{tab:mtconf}
\end{table}

\begin{table}
  \centering
  \begin{tabular}{|l|l|l|} \hline
    字段  & \cellalign{c}{名称} & \cellalign{|c|}{说明} \\ \hhline
    21:20 &         & 矩阵的元素大小: $2^{[21:20]}$ 字节 \\
    19:16 & Awcache & 写控制位。4'hF: 使用cache通路; 4'h0: uncache通路; 其它值无意义 \\
    15:12 & Awcmd   & 写控制位。Awcache为4'hF时，必须设为4'hB。否则无意义 \\
    11:8  & Arcache & 读控制位: 0xF时，cache通路;0x0，uncache 通路; 其它值无意义 \\
    7:4   & Arcmd   & 读控制位。Arcache为4'hF时，必须设为4'hC; 否则无意义 \\
    3     &         & 目标矩阵写入完毕后，是否有效中断 \\
    2     &         & 源矩阵读取完毕后，是否有效中断 \\
    1     &         & 是否允许写目标矩阵。为0时，转置过程只预取源矩阵，但不写目标矩阵 \\
    0     &         & 使能位 \\ \hline
  \end{tabular}
  \caption{trans\_ctrl 寄存器位域解释(其他位保留?)}
  \label{tab:mttransctrl}
\end{table}

\noindent \remark{问题：}
\begin{itemize}
  \item 源矩阵和目标矩阵的地址是否可以（部分）重合？
  \item Cache 一致性是否由硬件保证？
\end{itemize}
