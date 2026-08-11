# TX_RRH_Processor

> 所属层级：`MIMO_TX_Top` 直接子模块。
>
> 总索引：[MIMO_TX_Top子模块学习索引.md](./MIMO_TX_Top子模块学习索引.md)。
>
> 上一模块：[TX_RRH_FIFO_Exchange学习.md](./TX_RRH_FIFO_Exchange学习.md)。下一模块：[Over_Sample_Group学习.md](./Over_Sample_Group学习.md)。


## TX_RRH_Processor 顶层

### 实现功能

从四个天线对FIFO按无线帧和OFDM符号节拍读取数据，分别送入四条 `TX_RRH_Chain`，输出本 FPGA 对应的8路天线复数基带信号。

### 实现原理解读

`TX_Throttle` 根据源FIFO可读量和目标FIFO剩余空间决定何时启动；`Radio_Frame_Start_Generator` 形成一帧的符号时间线；`TX_FIFO_Read_Pattern_Generator` 给出具体读FIFO窗口。四条RRH链共享相同控制节拍，每条处理两根天线。

### 子模块关系图

```text
TX_RRH_Processor
├─ TX_Throttle：源/目标容量检查
├─ Radio_Frame_Start_Generator：帧和符号节拍
├─ TX_FIFO_Read_Pattern_Generator：FIFO读模式
└─ 4×TX_RRH_Chain
   ├─ Chain0 → antenna 0/1
   ├─ Chain1 → antenna 2/3
   ├─ Chain2 → antenna 4/5
   └─ Chain3 → antenna 6/7
```

### 关键代码解读

```verilog
TX_Throttle #(17543)
```

注意这里的17543与前级 `FEEDBACK_1/2=17443` 不同，二者相差100拍，不能混为同一个常量。

FIFO读请求设计为2拍延时：

```text
read_FIFO_flag
 → 延迟2拍得到data_in_valid
 → 与FIFO_data一起进入TX_RRH_Chain
```

### 讨论问题

1. `TX_RRH_Chain` 才是继续核实IFFT/WFRFT后时域整理、CP插入和天线数据成形的位置；顶层本身主要负责节流和四链并行。
2. 17443与17543的100拍差值应结合链内延时解释，不能先验地称为CP或保护间隔。
3. 只有Chain0的 `data_valid` 被接到顶层输出，设计默认四条链严格同步；需要仿真确认其他三链不会偏拍。
