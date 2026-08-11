# Over_Sample_Group

> 所属层级：`MIMO_TX_Top` 直接子模块。
>
> 总索引：[MIMO_TX_Top子模块学习索引.md](./MIMO_TX_Top子模块学习索引.md)。
>
> 上一模块：[TX_RRH_Processor学习.md](./TX_RRH_Processor学习.md)。这是当前发射端顶层链路的最后一个直接处理模块。


## Over_Sample_Group 顶层

### 实现功能

对8路天线I/Q基带样点进行缓存、时钟域适配、插零上采样和FIR插值滤波，再截位/饱和成16位DAC数据。

### 实现原理解读

每根天线实部、虚部分开处理，共16条标量通路：

```text
16-bit基带样点
 → FIFO缓存/跨时钟
 → 插零形成更高采样率序列
 → FIR_Filter_for_Upsampling
 → 32-bit滤波结果
 → over_under截位和溢出处理
 → 16-bit DAC样点
```

天线0..3使用同类同步FIFO，天线4..7使用带 `Asyn` 名称的FIFO，在 `clk` 与 `fmc2_dac_clk` 之间完成时钟适配。

### 子模块关系图

```text
Over_Sample_Group
├─ 8×real FIFO + 8×imag FIFO
├─ 有效窗口/插零控制
├─ 8×real FIR + 8×imag FIR
└─ 16×over_under：32位结果缩放为16位
```

### 关键代码解读

模块包含：

```text
16个输入FIFO
16个FIR_Filter_for_Upsampling
16个over_under定点截位模块
```

`write_zero[15:0]` 在有效无线帧窗口之外且FIFO未接近满时持续写0，使滤波器输入在空闲区保持定义良好的零序列。

### 讨论问题

1. 上采样倍率不能只从模块名推断，应结合 `counter/counter_asyn`、FIFO读使能和FIR输入valid波形确认。
2. `over_under` 的 `MSB=30、LSB=15` 表明从32位滤波结果截取定点窗口；是否带饱和、舍入还是直接截断需精读该子模块。
3. 当前大量full/empty异常检查逻辑被注释，正式上板前应确认FIFO不会在跨时钟启动或停帧时溢出/读空。
