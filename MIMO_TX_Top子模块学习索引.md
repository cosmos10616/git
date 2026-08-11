# MIMO_TX_Top子模块学习索引

> 组织原则：`MIMO_TX_Top` 的每一种直接子模块分别使用一份Markdown学习文档。三个 `TX_BIT_Processor` 实例结构相同，因此共用一份模块文档。

## 顶层处理链

```text
3×TX_BIT_Processor
 → TX_BIT_FIFO_Exchange
 → FIFO_Manager/Aurora跨板交换
 → TX_MIMO_Processor
 → TX_RRH_FIFO_Exchange
 → TX_RRH_Processor
 → Over_Sample_Group
 → DAC/RF
```

其中 `FIFO_Manager/Aurora` 位于更高一级的工程连接中，不是 `MIMO_TX_Top` 内部直接例化的子模块，因此暂不单独列为本顶层子模块文档。

## 模块文档

| 顺序 | MIMO_TX_Top直接子模块 | 顶层实例位置 | 学习文档 |
|---|---|---:|---|
| 1 | `TX_BIT_Processor`（3个实例） | 125、148、171 | [TX_BIT_Processor学习.md](./TX_BIT_Processor学习.md) |
| 2 | `TX_BIT_FIFO_Exchange` | 194 | [TX_BIT_FIFO_Exchange学习.md](./TX_BIT_FIFO_Exchange学习.md) |
| 3 | `TX_MIMO_Processor` | 221 | [TX_MIMO_Processor学习.md](./TX_MIMO_Processor学习.md) |
| 4 | `TX_RRH_FIFO_Exchange` | 263 | [TX_RRH_FIFO_Exchange学习.md](./TX_RRH_FIFO_Exchange学习.md) |
| 5 | `TX_RRH_Processor` | 352 | [TX_RRH_Processor学习.md](./TX_RRH_Processor学习.md) |
| 6 | `Over_Sample_Group` | 402 | [Over_Sample_Group学习.md](./Over_Sample_Group学习.md) |

## 学习顺序

```text
第一遍通读：
TX_BIT_Processor
 → TX_BIT_FIFO_Exchange
 → TX_MIMO_Processor
 → TX_RRH_FIFO_Exchange
 → TX_RRH_Processor
 → Over_Sample_Group

第二遍难点精读：
TTI/资源映射
 → Turbo/速率匹配
 → Gold加扰/调制
 → 12层到32天线的数据布局
 → RRH链/CP/上采样
```

## 待验证清单

- [ ] 恢复被乱码注释吞掉的 RTL 连接并通过语法检查；
- [ ] 用波形测量17443、17543和 `Trigger_Delay_10x` 的真实周期差；
- [ ] 核对 `Combine_Control_and_Data` 的3600有效子载波到4096点内部网格映射；
- [ ] 验证每子帧三次PDSCH transaction与9/13个PDSCH符号的完整闭合；
- [ ] 用波形验证 `TX_BIT_FIFO_Exchange` 的12拍调度、起始目标FPGA和64位数据格式；
- [ ] 精读 `TX_RRH_Chain` 内部的变换、CP插入和天线时域输出；
- [ ] 验证 `Over_Sample_Group` 的实际上采样倍率与定点缩放。

*最后更新：2026-08-11*
