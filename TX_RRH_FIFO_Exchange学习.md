# TX_RRH_FIFO_Exchange

> 所属层级：`MIMO_TX_Top` 直接子模块。
>
> 总索引：[MIMO_TX_Top子模块学习索引.md](./MIMO_TX_Top子模块学习索引.md)。
>
> 上一模块：[TX_MIMO_Processor学习.md](./TX_MIMO_Processor学习.md)。下一模块：[TX_RRH_Processor学习.md](./TX_RRH_Processor学习.md)。


## TX_RRH_FIFO_Exchange 顶层

### 实现功能

把 MIMO 处理输出的“按RB/子矩阵、覆盖多天线”的FIFO顺序，重新排列为 RRH 需要的“按天线组、覆盖全部RB”的四路FIFO顺序。

### 实现原理解读

输入分布按RB归属在四路外部FIFO中；模块根据 `RB_count` 选择当前应读哪一路，每个完整RB读取48个64位元素，最后一个不完整RB读取16个元素。写端再按两根天线一组分配到四个本地 RRH FIFO：

```text
FIFO1 → antenna 0/1
FIFO2 → antenna 2/3
FIFO3 → antenna 4/5
FIFO4 → antenna 6/7
```

### 子模块关系图

```text
4路MIMO FIFO
 → RB_count与FIFO余量判决
 → 48/16拍读取窗口
 → 数据重新排序
 → RRH_SYNC_Control跨板充足握手
 → 4×FIFO_for_TX_RRH_Buffer
 → 四个天线对FIFO
```

### 关键代码解读

RB与输入FIFO的当前对应关系：

```text
RB mod4 = 2 → 读FIFO1
RB mod4 = 0 → 读FIFO2
RB mod4 = 3 → 读FIFO3
RB mod4 = 1 → 读FIFO4
```

```verilog
N_input <= (last_partial_RB ? 16 : 48);
RB_count <= falling_edge ? ((RB_count==341) ? 0 : RB_count+1) : RB_count;
```

只有同步控制确认四路以及上游阶段数据充足时，才向后级报告可读总量16384。

### 讨论问题

1. 这是“存储布局转置”模块，不进行调制、预编码或滤波。
2. 342个RB是整个分布式交换网格的内部组织数量，不应直接当作LTE空口带宽RB数；它与四块FPGA、子矩阵和本工程自定义布局有关。
3. 四个输出FIFO的full端口未使用，安全性依赖固定生产/消费节拍和充足握手，需要波形验证。
