# MIMO 深度学习交流

> 逐模块、逐信号追踪 TX_BIT_Processor 九段流水线

---

## 1. TX_Trigger（第1级）

### 模块调用链
```
TX_Trigger.v
├── delay_1 (N=1) — 上升沿检测
└── TX_Symbol_Start_Generator.v
    ├── sync_latch_another — SR锁存器 (set优先)
    ├── Mod_N_Indexer (×2) — 符号周期计数器 + 帧符号数计数器
    ├── feedback_en_7_U16_bmp — 7槽循环选择器
    └── delay_1 (×2) — 输出打拍
```

### 关键信号流
```
radio_frame_trigger → 上升沿检测 → enable → sync_latch锁存
  → Mod_N_Indexer(symbol_durations, N=sym_dur=17443)
    → index_zero → symbol_start (每17443周期一个脉冲)
    → wrap_back → new_sym_dur (驱动7槽选择器 + 符号数计数器)
  → Mod_N_Indexer(symbols_per_radio_frame, N=140)
    → wrap_back → finish → 清零sync_latch (一帧结束)
```

### 核心参数

**140 = 1个无线帧的符号数**
```
1无线帧 = 10子帧 × 2时隙 × 7符号 = 140符号
```

**17443的推导**
```
基带采样率 = 61.44 MHz (FFT=4096, 是标准LTE的2倍)
时钟频率   = 245.76 MHz = 4 × 61.44 MHz
→ 每个基带样点需要 4 个时钟周期

一个OFDM符号体 = 4096样点 × 4周期/样点 = 16384周期 (纯数据)
17443 = 16384 + 1059 (流水线开销余量)

注意：CP(320/288样点)不在这里处理，由下游TX_RRH_Chain/CP_Insertion单独添加
```

**17443 和 140 的帧级验证**
```
17443 × 140 = 2,442,020 周期/帧
10ms × 245.76MHz = 2,457,600 周期/帧
剩余 = 15,580 周期 ≈ 63.4 μs → TDD保护间隔
```

### 时隙(Slot)在代码中的体现
- `OFDM_SYMBOL_PER_SLOT = 7` 已定义但Index_Generator状态机未使用
- `feedback_en_7_U16_bmp` 的7槽循环隐含了slot边界
- 当前FEEDBACK_1=FEEDBACK_2=17443，不区分长短CP

---

## 2. TTI_Handing_Top（第2级）

### 模块调用链
```
TTI_Handing_Top.v
├── Index_Generator.v (核心, 延迟4周期)
│   ├── sync_latch (clear优先SR锁存器)
│   ├── PULSE_TO_STROBE_U16 (脉冲→3600周期strobe + 0→3599子载波索引)
│   └── delay_n × 6
├── delay_en (N=1,W=64) — 帧起始锁存MCS_array
├── delay_1 (N=2) — PDCCH_trigger
└── delay_n (N=2,W=24) — 所有输出统一对齐
```

### Index_Generator核心逻辑
5个flag组成case选择码：`{flag_input, flag_Symbolindex1, flag_Symbolindex2, flag_Subindex1, flag_Subindex2}`

| Flag | 条件 | =1的含义 |
|------|------|----------|
| flag_input | iSymbolp==1 | 本周期有符号脉冲 |
| flag_Symbolindex1 | Sym<13 | 非子帧末符号 |
| flag_Symbolindex2 | Sym==0 | 子帧首符号 |
| flag_Subindex1 | Sub<9 | 非末子帧 |
| flag_Subindex2 | Sub==0 | 首子帧(子帧0) |

关键状态转移：
- `5'b11000~11110`: 子帧中间符号 → Symbindex++
- `5'b10010~10011`: 子帧最后符号(Sym=13) → Subp4=1, Sub++, Sym=0
- `5'b10000`: 无线帧最后符号(Sym=13,Sub=9) → 全部复位, Framep4=1
- `5'b11111`: 无线帧首符号(Sym=0,Sub=0) → 初始化处理(out1门控)

### out1的初始化作用
```
sync_latch(oSymbolp4) → out0 → delay_n(3) → out1
上电后前3周期out1=0，屏蔽索引递增
之后out1永久为1，正常计数
```

### CM_message结构
```
[111:88]  counter[23:0]      无线帧计数器
[87:84]   4'd10              子帧总数(固定)
[83:24]   MCS_array[59:0]    每子帧MCS配置
[23:0]    24'd0              保留
```

### 子载波strobe
```
每个oSymbolp4 → PULSE_TO_STROBE_U16(N=3600) → oCarrier=1持续3600周期
                                             → oCarindex 0→3599
```

### 延迟分析
```
总延迟 = 4(Index_Generator) + 2(delay_n) = 6周期
控制路径(6~13周期) vs 数据路径(20000周期)通过Trigger_Delay_10x对齐
```

### LabVIEW FPGA背景
Index_Generator的可读性问题源于它是LabVIEW FPGA图形化编程→Verilog自动翻译的产物：
- 5-bit case码是LabVIEW "多条件Case Structure"的直接翻译
- `sync_latch`是LabVIEW "Synchronous Latch"功能块的封装
- `PULSE_TO_STROBE_U16`注释明确写了"Labview的比较需要一个周期"
- 整个工作流: LabVIEW画图 → NI编译器 → Verilog → Vivado综合

---

## 3. TX_PDSCH_Configuration（第3级）

### 模块调用链
```
TX_PDSCH_Configuration.v
├── CM_to_PDSCH_Encoder.v (6周期) — MCS→modulation+TB_size
│   └── MCS_to_TBS.v (6周期) — 查表
├── Trigger_delay.v (×2) — ENCODE_SEG/ENCODE_SEG2延迟
│   └── PULSE_TO_STROBE_U32 — 脉冲→N周期strobe→下降沿检测
└── delay_n × 3 — 信号对齐
```

### 输入输出全景

**输入：**

| 信号 | 来源 | 含义 | 频率 |
|------|------|------|------|
| start_of_subframe | TTI_Handing_Top | 子帧边界脉冲 | 每1ms |
| start_of_symbol | TTI_Handing_Top | 符号起始脉冲 | 每~71μs |
| start_of_radio_frame | TTI_Handing_Top | 无线帧起始脉冲 | 每10ms |
| layer_ID | 顶层端口 | 层号(0-11) | 静态 |
| subframe_index | TTI_Handing_Top | 子帧号(0-9) | 每1ms变化 |
| MCS_array | CM_message[83:24] | MCS配置(实际未使用) | — |
| MCS_control | VIO手动开关 | 编码档位0/1/2/3 | 手动 |

**输出：**

| 信号 | 去向 | 含义 | 延迟 |
|------|------|------|:--:|
| PDSCH_transmitter_trigger | MAC_TX + PXSCH | 启动一次PDSCH编码 | +6 |
| OFDM_symbol_trigger | PXSCH_TX_Bit_Processing_Top | 符号节拍(数据路径对齐) | +7 |
| start_of_radio_frame_out | PXSCH_TX_Bit_Processing_Top | 帧边界参考 | +7 |
| modulation | PXSCH_TX_Bit_Processing_Top | QPSK=1/16QAM=2 | +5 |
| subframe_index_out | PXSCH_TX_Bit_Processing_Top | 子帧号(决定RE数) | +7 |
| transport_block_size | MAC_TX + PXSCH | TB大小(bit) | +5 |

### MCS_to_TBS查表

| MCS_control | 子帧0 | 其他子帧 |
|:--:|:---:|:---:|
| 0 | 7224 | 10296 |
| 1 | 8760 | 15840 |
| 2 | 15840 | 29296 |
| 3 | 8760 | 30576 |

- 子帧0 TB更小 → 因为PSS/SSS/PBCH占用了PDSCH的RE(10800 vs 15600)
- MCS_control=3时子帧0回退到8760 → 可能因为16QAM在子帧0干扰大
- 只有4档因为MCS_control是2-bit VIO开关
- MCS_array[59:0]虽然传入但未被使用

### PDSCH_transmitter_trigger的作用

**驱动两条路径：**
1. **MAC_TX.MAC_trigger** → 启动MAC_FIFO六状态机(IDLE→组帧→回IDLE)
2. **PXSCH_TX_Bit_Processing_Top.PXSCH_transmitter_trigger** → 锁存modulation/TB_size/RE_number/subframe参数

### ✅ Q1已解答：三段触发机制

```verilog
PDSCH_transmitter_trigger = temp | temp1 | temp2
// temp:  子帧边界+6
// temp1: 子帧边界+6+80130
// temp2: 子帧边界+6+160260
```

- **结论（2026-08-07验证）**：三段触发与码块分段**完全无关**。码块分段（C=1~5）在 CRC_Add_and_CB_Segmentation 状态机内部串行完成，不需要外部多次触发。三段触发的 OR 结构是为了支持比30576更大的TB——当TB大到编码管线超过80130周期时需要第二段触发。当前max_TB=30576只用第一段，temp1/temp2是继承自更大带宽参考设计的遗留机制。
- 验证来源：精读 PXSCH_Parameter_Computation（8个TB_size查表，C=1~5）+ CRC_Add_and_CB_Segmentation（五状态机内部处理所有码块分段）

---

## 4. Buffer_UDP_Data（第4级）— 2026-08-06 精读

### 模块定位

UDP侧时钟(`clk_udp`)与基带时钟(`clk_baseband=245.76MHz`)之间的异步FIFO缓冲。薄封装层，核心是Xilinx FIFO Generator IP。

### 模块调用链
```
Buffer_UDP_Data.v（薄封装层）
└── FIFO_Buffer_UDP (Xilinx FIFO Generator 13.2 IP核)
```

### 关键信号流

```
UDP侧(clk_udp):    udp_input_valid + udp_input_data[7:0] → FIFO写端口
基带侧(clk_baseband): FIFO读端口 → data_valid + data_out[7:0] → MAC_FIFO
```

**核心握手（FWFT模式下的反压握手）：**
```verilog
data_valid = fifo_valid & ask_for_data
// fifo_valid: FIFO非空时自动拉高，数据已预置在dout总线（FWFT特性）
// ask_for_data = MAC_ready_for_data = MAC_FIFO.read_FIFO_flag
```

**闭环流控链路：**
```
MAC_FIFO需要数据 → read_FIFO_flag=1 → ask_for_data=1
  → data_valid=1 (FIFO非空时) → MAC_FIFO锁存数据
MAC_FIFO暂停 → read_FIFO_flag=0 → ask_for_data=0
  → data_valid=0 → FIFO不读出 → 可能满 → full=1 → UDP侧停写
```

### FIFO IP 关键参数

| 参数 | 值 | 含义 |
|------|-----|------|
| 数据宽度 | 8-bit | 字节流 |
| 深度 | 65536 | 64KB Block RAM |
| 时钟模式 | Independent Clocks | 异步FIFO（读写时钟独立） |
| 存储类型 | Block RAM | 非分布式RAM |
| 输出模式 | FWFT | First Word Fall Through（零延迟读） |
| rd_data_count 宽度 | 17-bit | 供MAC_FIFO判断可用数据量 |
| Safety Circuit | 使能 | 复位安全保护 |
| 同步器级数 | 2 | 跨时钟域同步 |

### FWFT 模式优势

标准FIFO需要先拉rd_en，下一周期数据才出现在dout。FWFT模式下：
- 数据提前出现在dout
- rd_en与data_valid同一周期
- 无读取延迟，适合高速流水线

### 深度选择理由

64KB Block RAM缓冲平滑两方面的速率不匹配：
- UDP突发流量（网络抖动） vs 基带匀速处理（每子帧固定bit数）
- 跨时钟域（clk_udp vs 245.76MHz）

### fifo_read_count 的作用

MAC_FIFO用此计算 `remaining_FIFO_elements = min(TB_size/8 - 4, used_FIFO_depth)`：
- `TB_size/8 - 4`：理论需要字节数（减4是帧头32bit占4字节的裕量）
- `used_FIFO_depth`：FIFO实际可读字节数
- min() 防止读空——如果UDP侧发得慢，FIFO不够TB_size，就只发实际有的数据

### 反压传播链（五级传播）

```
基带忙 → ready_for_output=0 → MAC暂停 → read_FIFO_flag=0
  → FIFO不读 → 可能满 → full=1 → UDP侧停写
```

---

## 5. MAC_TX / MAC_FIFO（第5级）— 2026-08-07 精读

### 模块定位

将8-bit字节流组MAC帧，转换为串行bit流输出给Turbo编码器。时钟245.76MHz（端口名`clk_192M`是历史遗留）。

### 模块调用链
```
MAC_TX.v（58行，薄封装层）
└── MAC_FIFO.v（244行，核心状态机）
    ├── delay_n #(N=1, Width=33) — output_length延迟1周期
    ├── FIFO_MAC_handshake (Xilinx FIFO IP) — 1-bit宽FWFT FIFO，平滑输出
    └── 六状态机
```

### MAC_TX 薄封装层

仅实例化 MAC_FIFO，纯连线：
```verilog
MAC_trigger      = PDSCH_transmitter_trigger  // 来自TX_PDSCH_Configuration
ready_for_output = PXSCH_ready                // 来自Turbo编码器反压
TB_size          = {15'd0, transport_block_size}
```

`last` 输出悬空（未实现，MAC_FIFO 中 last_sample_out_reg 赋值全部注释掉）。

### MAC_FIFO 子模块

| 子模块 | 类型 | 功能 |
|--------|------|------|
| `delay_n #(N=1, Width=33)` | 通用移位寄存器 | `{output_length_valid, output_length[31:0]}` 延迟1周期，补偿FIFO读延迟 |
| `FIFO_MAC_handshake` | Xilinx FIFO IP | 1-bit宽 FWFT FIFO，平滑状态机间隙输出（输出侧有时发有时不发） |

### 六状态机详解

```
IDLE(0) → COMPUTE(1) → SEND_HEADER(2) ⇄ READ_FIFO(4) ⇄ SEND_PAYLOAD(3) → SEND_PADDING(5) → IDLE
```

**状态 0 — IDLE：**
- 等待 `output_length_valid_delay1`（=PDSCH_transmitter_trigger 延迟1周期）
- 计算初始参数：
  - `padding_bits = TB_size - 32`（先假定全部填充零，后面再修正）
  - `remaining_FIFO_elements = min(TB_size/8 - 4, used_FIFO_depth)`（受限于FIFO实际可用量，防读空）

**状态 1 — COMPUTE_REMAINING_CONFIGS（1周期）：**
- `padding_bits = padding_bits - (remaining_FIFO_elements × 8)` → 真正的填充bit数
- 例：TB=288bit，FIFO有20字节 → padding = (288-32) - 160 = 96bit
- 无条件跳转 SEND_HEADER

**状态 2 — SEND_HEADER（32周期）：**
- `bit_out_reg = remaining_FIFO_elements[bit_counter]` — LSB first 发32-bit帧头
- 帧头 = payload字节数（接收端据此知道要收多少字节）
- 最后一bit时预拉高 `read_FIFO_flag` — 预读第一个payload字节

**状态 4 — READ_FIFO（1周期）：**
- `read_FIFO_flag = 0`（只持续1周期脉冲）
- `remaining_FIFO_elements--`
- `valid_out_reg = 0`（本周期只读不发）
- 与 FWFT FIFO 的时序：T-1周期拉高rd_en → T周期数据出现在dout → T周期锁存FIFO_data

**状态 3 — SEND_PAYLOAD（8周期）：**
- `bit_out_reg = FIFO_data[bit_counter]` — LSB first 逐bit发送
- 最后一bit时预拉高 `read_FIFO_flag` — 预读下一字节
- 退出决策：
  - remaining>0 → READ_FIFO（继续读）
  - remaining=0 & padding>0 → SEND_PADDING
  - 都=0 → IDLE

**状态 5 — SEND_PADDING（可变周期）：**
- `bit_out_reg = 0` — 全部发零
- `bit_counter++`，直到 `bit_counter_add1 >= padding_bits` → IDLE

### 关键设计要点

**bit_counter / bit_counter_add1 双寄存器：**
`bit_counter_add1` 始终领先 `bit_counter` 1个值，用于 Next_state 逻辑提前一周期预判"下一拍是否完成"。

**FWFT两次握手机制：**
read_FIFO_flag 在发完前一单元的最后一bit时预拉高，利用 FWFT 的零延迟特性，数据同周期就绪。

**帧格式：**
```
|← 32-bit头(LSB first) →|← N字节payload(每字节LSB first) →|← M bit填充零 →|
```

**反压链路：**
```
PXSCH_ready=0 → valid_out门控为0 → 状态机暂停
  → read_FIFO_flag=0 → ask_for_data=0
  → Buffer_UDP_Data停止读出 → 可能满 → full=1 → UDP停写
```

**FIFO欠载保护：**
`remaining_FIFO_elements = min(需要, 实际可用)` 防止读空——UDP发得慢时不会死等。

**edge case：**
TB<8bit 时跳过所有payload，直接发帧头（全0数据长度）然后填充零。

### 完整帧时序（TB=288bit, FIFO充足）

```
周期 0:     IDLE        收到trigger延迟
周期 1:     COMPUTE     padding=0
周期 2-33:  SEND_HEADER 发32bit帧头，第33周期预读FIFO
周期 34:    READ_FIFO   读第1字节
周期 35-42: SEND_PAYLOAD 发第1字节8bit，第42周期预读
...循环（34→35-42→34）共32次...
周期 298:   IDLE        帧结束
```

总322周期发288bit，有效吞吐 ~0.89 bit/cycle（约219 Mbps @ 245.76MHz）。

---

### 💬 讨论：关于MAC模块的几个关键问题

#### Q1: 为什么MAC的数据要逐bit输出？

MAC_FIFO 把8-bit字节拆成1-bit串行输出，看起来是"降速"，实际原因是**整个下游处理链都是按 bit 为单位设计的**。

**根本原因：Turbo 编码器是逐 bit 的状态机**

Turbo 编码的核心是 RSC（递归系统卷积）编码器：

```
bit_in → [异或] → [移位寄存器0] → [移位寄存器1] → [移位寄存器2] → ...
              ↑                        ↓
              └──────── [反馈] ────────┘
```

每个输入 bit 进入编码器后，和内部状态（3个移位寄存器）做异或，产生输出，同时状态迁移到下一值。这是一个**严格串行的状态机**——你不能一次丢 8 个 bit 进去，因为第 2 个 bit 的处理依赖第 1 个 bit 处理后的新状态。

```verilog
// Turbo_Encoder 内部：一个周期只处理一个 bit
num_encoded_bits <= num_encoded_bits + 1'b1;  // 逐bit计数
```

**整个下游链路都是 bit 级的**

从 MAC_FIFO 的输出开始，整条链都是 1bit/cycle 的节奏：

```
MAC_FIFO → CRC_Adder → Turbo_Encoder → Rate_Matcher → Scrambler → Bit_to_Symbol
  1bit       1bit         1bit           1bit           1bit        (这里才转并行)
```

每一步都是 bit 级操作：
- **CRC**：LFSR 逐 bit 计算，当前 CRC 值取决于上一个 bit 算完的状态
- **Turbo**：如上所述，状态机
- **速率匹配**：子块交织器把 bit 按32列矩阵重排，操作粒度是单个 bit
- **加扰**：Gold 序列逐 bit 异或

直到 **Bit_to_Symbol** 才把串行 bit 拼成并行符号（2/4/6 bit），因为调制器才需要并行输入。

**如果 MAC_FIFO 输出8-bit并行会怎样？**

那在进入 CRC 之前就要加一个并转串模块，本质上只是把串行化推迟了一级，没有任何好处，反而多了：
- 字节内 bit 顺序问题（MSB first 还是 LSB first）
- Turbo 编码器前面还要加 buffer
- CRC 计算需要支持并行输入（更复杂的组合逻辑）

**速度够吗？**

245.76MHz × 1bit/cycle = **245.76 Mbps** 原始速率。

一个子帧（1ms）最大 TB 才 30576 bit，加上 CRC 和编码开销约 10 万 bit，实际需要的速率只有 ~100 Mbps——1bit/cycle 的串行流绰绰有余，根本不是瓶颈。

**一句话总结：** 不是 MAC 选择了串行输出，而是 **Turbo 编码器只能串行吃数据**，整条链自然就是串行的。MAC 的字节→bit 转换只是把这个必然发生的串行化提前做了而已。

---

#### Q2: MAC模块具体做什么？是标准的MAC层协议吗？

MAC_FIFO 的核心工作就一件事：**把 FIFO 里的字节数据，加上帧头，转成固定长度的串行 bit 流**。

**帧格式：**

```
┌────────────────────┬──────────────────────────────┬──────────────────┐
│   32-bit 帧头       │    N 字节 payload             │   M bit 零填充    │
│  (payload字节数)    │   (从FIFO读出的原始数据)        │   (凑够TB_size)   │
│   LSB first         │   每字节 LSB first             │                  │
└────────────────────┴──────────────────────────────┴──────────────────┘
                     ←────────── TB_size bit ──────────→
```

帧头只有32-bit，内容是 **payload的字节数**（不是bit数）：

```verilog
// SEND_HEADER 状态:
bit_out_reg <= remaining_FIFO_elements[bit_counter];
// 把 remaining_FIFO_elements（payload字节数）按 LSB first 逐bit发出
```

**六状态机具体流程（以 TB_size=288（36字节）、FIFO 里有 20 字节为例）：**

```
IDLE (等trigger)
  │ output_length_valid=1, output_length=288
  ↓
COMPUTE (1周期)
  │ padding_bits = 288 - 32 = 256     (先假设全是填充)
  │ remaining_FIFO_elements = min(288/8-4, 20) = min(32, 20) = 20
  ↓
SEND_HEADER (32周期)
  │ 逐bit发 20 (payload字节数): 00101000 00000000 ...  (32bit, LSB first)
  │ 最后一bit时预拉 read_FIFO_flag，提前读第一字节
  ↓
READ_FIFO (1周期)  ←──────────┐
  │ 读一个字节，不发送          │
  ↓                            │
SEND_PAYLOAD (8周期)           │ 循环20次
  │ 逐bit发这个字节(LSB first)  │
  │ 最后一bit时预拉 read_FIFO_flag│
  ↓                            │
  remaining>0 → READ_FIFO ─────┘
  remaining=0, padding>0 → SEND_PADDING
  ↓
SEND_PADDING (可变周期)
  │ 发 zero bit，直到 bit_counter >= padding_bits
  ↓
IDLE
```

**这和标准 MAC 层协议一致吗？**

**完全不是标准 LTE MAC 层**。标准 LTE MAC 层非常复杂：

| 功能 | 标准 LTE MAC | 这个 MAC_FIFO |
|------|:--:|:--:|
| 调度/优先级 | ✅ 复杂的逻辑信道优先级 | ❌ 没有 |
| HARQ 重传 | ✅ 8进程HARQ | ❌ 没有 |
| 复用 | ✅ 多个逻辑信道复用到一个传输块 | ❌ 没有 |
| MAC PDU 格式 | R/E/LCID/F/L 子头 + MAC SDU + padding | ❌ 只有长度+数据+零填充 |
| 加密 | ✅ 控制面加密 | ❌ 没有 |

这个"MAC"其实就是一个**自定义的成帧器（Framer）**——它解决的是这个特定 FPGA 系统里一个非常实际的问题：

> UDP 发来的数据是按字节打包的，Turbo 编码器是按 bit 吃的，而且每个子帧要发固定 TB_size 个 bit。如果 UDP 数据不够 TB_size，就得填零；如果 UDP 数据比 TB_size 多，就得等下一个子帧。

**帧头的唯一作用**：告诉接收端"我实际发了多少个字节的有效数据"。因为 TB_size 是双方约定的，但实际有效数据可能小于 TB_size，接收端需要知道 padding 从哪开始，才能正确去掉零填充、恢复原始 UDP 包。

**为什么叫 MAC？** 因为它在系统中的位置"像"LTE 的 MAC 层——UDP（应用层）→ MAC（数据链路层）→ PHY（物理层）。但这个命名容易误导，更准确的名字应该是 `Data_Framer` 或 `Stream_Bridge`。

---

#### Q3: "循环20次"是什么意思？

"20"指的是上面例子中 `remaining_FIFO_elements = 20`（FIFO 里有 20 个字节）。状态机每次循环处理 1 个字节，所以循环 20 次。

**循环的具体内容：**

```
                        ┌──────────────────────────┐
                        │     READ_FIFO (1周期)      │
                        │  读1个字节到 FIFO_data     │
                        │  remaining_FIFO_elements-- │
                        └──────────┬───────────────┘
                                   ↓
                        ┌──────────────────────────┐
                        │   SEND_PAYLOAD (8周期)     │
                        │  把这1个字节逐bit发出去     │
                        │  发完第8bit时判断:          │
                        │    remaining>0? ──→ 继续循环 │
                        │    remaining=0? ──→ 退出    │
                        └──────────┬───────────────┘
                                   │
                    remaining>0    │
                    回到 READ_FIFO ←┘
```

**代码上看就是两个状态的互相跳转：**

```verilog
// SEND_PAYLOAD 的下一状态判断（MAC_FIFO.v 第118行）:
SEND_PAYLOAD : Next_state = (ready_for_output & (bit_counter_add1 >= 32'd8)) ?
    // 一个字节(8bit)发完了，还剩下字节吗？
    ((remaining_FIFO_elements != 32'd0) ? READ_FIFO :     // ← 有剩余，回去再读一个字节
     ((padding_bits > 32'd0) ? SEND_PADDING : IDLE))       // ← 没了，去填充或结束
    : Current_state;  // 还没发完8bit，继续

// READ_FIFO 无条件跳到 SEND_PAYLOAD（MAC_FIFO.v 第120行）:
READ_FIFO : Next_state = SEND_PAYLOAD;
```

所以 `SEND_PAYLOAD ⇄ READ_FIFO` 形成一个两状态的小循环，每 9 个周期处理 1 个字节（1周期读 + 8周期发）。FIFO 里有 20 个字节就循环 20 次，有 100 个就循环 100 次。

**"20"不是固定值**，它等于 `min(TB_size/8 - 4, used_FIFO_depth)`——取"需要的字节数"和"FIFO实际有的字节数"中较小的那个，防止 FIFO 数据不够时读空。

---

#### Q4: SEND_PAYLOAD 逐bit发的数据到底从哪来？

从 SEND_PAYLOAD 这个代码行追溯：

```verilog
// MAC_FIFO.v 第177行
bit_out_reg <= FIFO_data[bit_counter];
//             ↑          ↑
//        8-bit寄存器   0→7 (发完一个字节归零)
```

`FIFO_data` 是一个 8-bit 寄存器，存的是**当前正在发送的那个字节**。那这个字节从哪来？

**数据来源的完整链条：**

```
UDP网络包
  ↓ (clk_udp时钟域)
FIFO_Buffer_UDP (64KB 异步FIFO)
  ↓ (clk_baseband时钟域, 245.76MHz)
Buffer_UDP_Data 模块
  │ 输出: data[7:0] + data_valid
  ↓
MAC_FIFO
  │ data[7:0] 端口 ← 就是 Buffer_UDP_Data 的输出
  │
  ├─ READ_FIFO 状态: read_FIFO_flag=1 → 通知上游"给我一个字节"
  │     ↓
  │  上游 data_valid=1 时, FIFO_data <= data[7:0]  ← 锁存这个字节
  │
  └─ SEND_PAYLOAD 状态: FIFO_data[bit_counter] → 逐bit输出
        bit_counter=0 → 发 bit0 (LSB first)
        bit_counter=1 → 发 bit1
        ...
        bit_counter=7 → 发 bit7, 然后归零, 进入 READ_FIFO 再读下一字节
```

**关键：read_FIFO_flag 怎么把数据"拉"过来的：**

```verilog
// MAC_FIFO.v 第222-234行
always @(posedge clk) begin
    if(FIFO_out_valid)        // ← Buffer_UDP_Data 说"数据有效"
        FIFO_data <= data;    // ← 锁存新的8-bit字节
    else
        FIFO_data <= FIFO_data;  // 保持不变，继续发当前字节
end
```

`read_FIFO_flag` 拉高 → Buffer_UDP_Data 收到后读 FIFO → 下一个周期 `FIFO_out_valid=1` → `FIFO_data` 锁存新字节。

**注意时序**：`read_FIFO_flag` 是在 SEND_PAYLOAD 发最后一个 bit（第7bit）时拉高的，利用 FWFT FIFO 的零延迟特性，下一周期（READ_FIFO）新字节就已经出现在 `data[7:0]` 上了。

所以 SEND_PAYLOAD 里 `FIFO_data[bit_counter]` 读的其实是**上一轮 READ_FIFO 状态锁存的那个字节**。进 READ_FIFO 只做一件事——把下一个字节准备好，然后立刻回 SEND_PAYLOAD 开始逐bit发送。

---

### ⚡ 流程串讲：MAC 一帧的完整生命周期

把上面四个问题串起来，从 MAC_FIFO 的视角看一个帧的完整处理过程：

**1. 等米下锅 (IDLE)**

万事俱备，等着。`output_length_valid`（= PDSCH_transmitter_trigger 延迟1周期）来的那一刻：
- 拿到 TB_size（比如288bit = 36字节）
- 看一眼 FIFO 里有多少数据（比如20字节）
- 算出 padding = 288 - 32(帧头) - 20×8(实际数据) = 96bit 零填充

**2. 先报数 (SEND_HEADER, 32周期)**

把"我实际有20字节"这个数，按 LSB first 方式，一个bit一个bit发出去。
接收端收到这32bit，就知道后面要收20字节有效数据 + 若干零填充。

**3. 搬砖 (SEND_PAYLOAD ⇄ READ_FIFO, 每字节9周期)**

开始真正的数据搬运：
- READ_FIFO（1周期）：向 Buffer_UDP_Data 要一个字节 → 锁存到 `FIFO_data`
- SEND_PAYLOAD（8周期）：把 `FIFO_data` 的8个bit，从 LSB 开始一个一个发出去
- 发完回到 READ_FIFO 要下一个字节
- 20个字节，来回20趟

**4. 凑数 (SEND_PADDING, 96周期)**

20字节发完了，但承诺要发288bit。34字节×8=272bit，还差96bit。
剩下的96个周期全部发0——这些是padding，接收端知道（因为帧头说了只有20字节），会直接丢掉。

**5. 收工 (IDLE)**

所有bit发完，状态机回到 IDLE，等下一个子帧的 trigger。

---

**整个过程的流水线视图：**

```
周期:  0        1         2-33           34    35-42     43    44-51   ...  290-297   298
状态: IDLE → COMPUTE → SEND_HEADER → READ_FIFO SEND_PAYLOAD READ_FIFO SEND_PAYLOAD ... SEND_PADDING → IDLE
                                           ↑        ↑        ↑        ↑
                                           │        │        │        │
                                     读第1字节  发第1字节  读第2字节  发第2字节  ...  发96bit零填充
                                     锁存FIFO_data  逐bit发  锁存新字节  逐bit发
```

`read_FIFO_flag` 的预拉高时机：每次 SEND_PAYLOAD 发到第7bit（最后一bit）时拉高，下一周期 READ_FIFO 就能看到新数据。这个时序利用了 FWFT FIFO 的特性——rd_en 和 data 同周期有效，中间没有等待。

---

#### Q5: output_length_div8、output_length_div8_minus4、output_length_minus32 这三个变量的单位，为什么可以放在同一个等式里？

这个问题来自 MAC_FIFO 里的这行代码：

```verilog
remaining_FIFO_elements_temp = (output_length_div8 != 32'd0) ? output_length_div8_minus4 : output_length_div8;
```

这个等式里的三个变量其实**单位完全一致，都是字节**。来看源码中这三个变量是怎么算的：

```verilog
// MAC_FIFO.v 第239-241行
output_length_div8        <= (output_length >> 2'd3);               // ①
output_length_div8_minus4 <= (output_length >> 2'd3) - 3'd4;       // ②
output_length_minus32     <= output_length - 6'd32;                 // ③
```

逐行解释：

**① output_length_div8 —— 整帧有多少字节**

`output_length` 就是 TB_size，单位是 **bit**。右移 3 位等于除以 8，把 bit 转成 byte。

```
例：TB_size = 288 bit → output_length_div8 = 288/8 = 36 字节
```

**② output_length_div8_minus4 —— payload 有多少字节（减去帧头）**

同样是 TB_size/8（字节），然后减 4。这个 **4 是 4 个字节**，对应 32-bit 的帧头。

```
例：TB_size = 288 bit → output_length_div8_minus4 = 288/8 - 4 = 36 - 4 = 32 字节
```

帧头占 32 bit = 4 字节，正负载荷 = 36 - 4 = 32 字节。

**③ output_length_minus32 —— 整帧去掉帧头后还剩多少 bit**

这是同一个概念，但用**bit**做单位。32 = 32 bit，即帧头大小。

```
例：TB_size = 288 bit → output_length_minus32 = 288 - 32 = 256 bit
```

**三个变量的对照：**

| 变量 | 单位 | TB=288时的值 | 含义 |
|------|:--:|:--:|------|
| `output_length` (TB_size) | bit | 288 | 整帧总大小 |
| `output_length_div8` | **字节** | 36 | 整帧 = 36 字节 |
| `output_length_div8_minus4` | **字节** | 32 | payload = 36-4 = 32 字节 |
| `output_length_minus32` | **bit** | 256 | payload = 288-32 = 256 bit |

32 字节 × 8 = 256 bit，对得上。

**回到那句代码：**

```verilog
remaining_FIFO_elements_temp = (output_length_div8 != 32'd0) ? output_length_div8_minus4 : output_length_div8;
//                                       ↑ 36≠0 为真          ↑ 取 32 字节
```

翻译成人话：
- 如果整帧有数据（TB_size/8 ≠ 0），我需要从 FIFO 读的字节数 = payload 的字节数 = TB_size/8 - 4
- 如果整帧没数据（TB_size=0 的极端情况），那就读 0 字节

`output_length_div8`（36 字节）和 `output_length_div8_minus4`（32 字节）单位完全相同，都是字节，放在同一个等式里没有任何问题。

**容易混淆的地方：**

`minus4` 和 `minus32` 看起来像是"减 4 bit"和"减 32 bit"，但实际上：
- `minus4` → 减 4 **字节** = 32 bit（帧头大小）
- `minus32` → 减 32 **bit** = 4 字节（帧头大小）

它们是同一个东西——帧头的长度——只是用不同单位表达。两个变量不会同时出现在一个等式里：`output_length_div8_minus4`（字节）用在字节计数场景，`output_length_minus32`（bit）用在 bit 计数场景（比如算 padding_bits）。

**这两个变量在哪里使用，验证了单位的一致性：**

```verilog
// 字节场景：和 used_FIFO_depth（也是字节）比较
remaining_FIFO_elements <= (remaining_FIFO_elements_temp < used_FIFO_depth) 
                           ? remaining_FIFO_elements_temp : used_FIFO_depth;

// bit场景：算需要填充多少bit
padding_bits <= (output_length_div8 != 32'd0) ? output_length_minus32 : output_length_delay1;
```

`output_length_div8_minus4`（字节）≈ `output_length_minus32 / 8`（bit ÷ 8 = 字节），一个给字节计数器用，一个给 bit 计数器用，分工明确。

## 6. PXSCH_TX_Bit_Processing_Top（第6级）— 2026-08-07 精读 ★核心DSP

### 这个模块是干什么的？

一句话：**把 MAC_FIFO 发来的一串 bit，变成 LTE 的 I/Q 频域符号**。

输入是一根串行bit线（1bit/cycle），输出是一对16-bit的I/Q复数符号（fix16_15定点格式）。中间要经历：Turbo编码(1/3码率) → 速率匹配(打孔/重复) → Gold序列加扰 → QPSK/16QAM/64QAM星座映射。

打个比方：MAC_FIFO给的是"原材料"（用户的原始bit），这个模块是"加工厂"，产出的是"标准件"（LTE协议要求的调制符号），下一级 Combine_Control_and_Data 负责把这些标准件装到 OFDM 资源网格的正确位置。

### 顶层接口——每个信号的含义

```
                          PXSCH_TX_Bit_Processing_Top
                         ┌──────────────────────────────┐
   clk_192M (245.76MHz) ─→│                              │
   rst_n ────────────────→│                              │
                         │                              │
   MCS_data ─────────────→│  串行bit输入（来自MAC_FIFO）    │──→ ready_for_input (反压上游)
   MCS_valid ────────────→│                              │
                         │                              │
   PXSCH_transmitter_    │  触发脉冲：启动一次编码          │──→ symbol_valid
    trigger ─────────────→│  （↑上升沿锁存所有配置参数）      │──→ symbol_real[15:0]
                         │                              │──→ symbol_imag[15:0]
   OFDM_symbol_trigger_in→│  OFDM符号节拍（延迟匹配用）      │──→ OFDM_symbol_trigger_out
                         │                              │
   modulation[1:0] ──────→│  1=QPSK  2=16QAM  3=64QAM   │
   subframe_index[3:0] ──→│  子帧号 0~9                  │
   transblock_size[16:0] ─→│  TB大小（bit数）              │
   delay[31:0] ──────────→│  触发延迟值（=20000）          │
                         └──────────────────────────────┘
```

**输入详解：**

| 信号 | 来源 | 含义 | 为什么需要 |
|------|------|------|-----------|
| MCS_data | MAC_FIFO 的 PXSCH_bit | 串行bit流，MSB first | Turbo编码器需要逐bit输入 |
| MCS_valid | MAC_FIFO 的 PXSCH_bit_valid | bit有效标志 | 区分有效数据和空闲周期 |
| PXSCH_transmitter_trigger | TX_PDSCH_Configuration | 一次传输的启动脉冲（每子帧一个） | 锁存参数+启动CRC分段状态机 |
| OFDM_symbol_trigger_in | TX_PDSCH_Configuration | 每个OFDM符号起始脉冲 | 经过20000周期延迟后输出，与编码后的符号对齐 |
| modulation | TX_PDSCH_Configuration | 调制方式 | 决定bits_per_symbol(2/4/6)和星座映射 |
| subframe_index | TTI_Handing_Top | 子帧号 | 决定RE数量（子帧0=10800，其他=15600），还参与c_init计算 |
| transblock_size | TX_PDSCH_Configuration | TB的bit数 | 决定码块数C、码块大小K等全部编码参数 |
| delay | TX_PDSCH_Configuration | 20000 | 编码管线的总延迟，用于OFDM触发对齐 |

**输出详解：**

| 信号 | 去向 | 含义 |
|------|------|------|
| ready_for_input | MAC_FIFO | 反压信号——编码管线忙时拉低，MAC_FIFO暂停发bit |
| symbol_valid | Combine_Control_and_Data | I/Q符号有效 |
| symbol_real[15:0] | Combine_Control_and_Data | 实部，fix16_15格式（1位符号+1位整数+14位小数） |
| symbol_imag[15:0] | Combine_Control_and_Data | 虚部，同上 |
| OFDM_symbol_trigger_out | Combine_Control_and_Data | 延迟20000周期后的符号触发 |

---

### 整体设计思路——两大分支

这个模块的顶层设计非常清晰：**两条串行流水线并行工作，最后汇合**。

```
   MCS_data ──→ ┌─────────────────────────────────┐
   (串行bit)     │  分支1: PXSCH_Channel_Encoder    │
   MCS_valid ──→│  (编码链)                        │──→ encoding_bit_out
                │  bit→参数查表→CRC→Turbo→速率匹配    │    (编码后的串行bit)
   trigger ────→│                                  │
   modulation ─→│                                  │
   TB_size ────→└─────────────────────────────────┘
                                                    │
                                                    ↓
                ┌─────────────────────────────────┐
                │  分支2: PXSCH_Deserializer_      │
                │  Scrambler_Modulator (加扰调制链) │──→ symbol_real/imag
                │  bit→串并转换→加扰→星座映射        │    (I/Q fix16_15)
   trigger ────→│                                  │
   modulation ─→│                                  │
   subframe_idx→│                                  │
   cell_ID ────→└─────────────────────────────────┘
```

**设计思想：**

1. **全串行处理**：bit一个一个进来，最终一个一个变成符号。不需要并行总线，硬件开销小。
2. **参数预计算+锁存**：trigger 到来时锁存 modulation/TB_size/subframe_index，在整个编码过程中这些参数保持不变，防止中途变化导致编码错乱。
3. **延迟匹配**：编码链的延迟是可变的（取决于TB大小），但 OFDM 符号触发需要精确对齐。用 `Trigger_Delay_10x` 给触发信号加固定延迟（20000周期），保证"触发"和"数据"同步到达下游。
4. **反压传播**：从最下游（速率匹配器）向上游（MAC_FIFO）逐级反压，任何一级忙都能让整条流水线停下来。

---

### 顶层代码的实现细节

**参数锁存怎么做的？**

```verilog
// 在 PXSCH_transmitter_trigger 上升沿锁存所有配置
always @(posedge clk_192M) begin
    if(~rst_n)
        modulation_delay1 <= 2'd1;          // 默认QPSK
    else if(PXSCH_transmitter_trigger)      // ← 只有trigger脉冲时才更新
        modulation_delay1 <= modulation;    // 锁存调制方式
    else
        modulation_delay1 <= modulation_delay1;  // 其他时间保持不变
end
```

为什么不用 `PXSCH_transmitter_trigger` 直接连？因为 trigger 只持续1个周期，但编码过程持续几万个周期。在这期间 modulation 引脚可能已经变了（下一个子帧的配置来了），但当前编码必须用老的参数。

**两条分支是怎么连接的？**

```verilog
// 分支1输出 → 分支2输入（只用了3根线）
PXSCH_Deserializer_Scrambler_Modulator (
    .bit_in(encoding_bit_out),           // 编码后的bit
    .valid_in(encoding_bit_out_valid),   // 编码bit的有效信号
    .new_control_pulse(PXSCH_transmitter_trigger_delay1),  // 触发加扰初始化
    .modulation(modulation_delay1),      // 同一个锁存值
    // ...
);
```

非常简单：编码链输出的串行bit + valid，直接连到加扰调制链的输入。注意 trigger 被 `delay_1` 延迟了1周期——因为编码链的 `PXSCH_transmitter_trigger_delay1` 晚了1拍。

**输出为什么又打了1拍？**

```verilog
delay_1 #(1) delay_PXSCH_TX_modulation_valid(
    .In(modulation_valid), .Out(symbol_valid));
delay_n #(1,32) delay_PXSCH_TX_modulation_symbol(
    .In({modulation_imag, modulation_real}), .Out({symbol_imag, symbol_real}));
```

这是 FPGA 设计的标准做法：输出寄存器化，改善时序收敛。顶层输出直接来自寄存器，组合逻辑路径最短。

---

### 分支1：PXSCH_Channel_Encoder — 编码链

#### 子模块连接关系

```
                    ┌──────────────────────────────────────────────────┐
                    │            PXSCH_Channel_Encoder                 │
                    │                                                  │
  bit_in ──────────┐                                                  │
  bit_in_valid ────┤                                                  │
                    │                                                  │
  configuration ────┼──→ [PXSCH_Parameter_Computation] (98周期)        │
  TB_size ──────────┘    │ C, K, Kw, E+, E-, k0, ... (166-bit参数总线) │
                         │ ready_for_configuration                     │
                         ↓                                             │
                    [CRC_Add_and_CB_Segmentation] (五状态机)            │
                    bit_in→│TB_CRC(24A)→CB_CRC(24B)→bit_out            │
                           │ready_for_input(→上游MAC_FIFO)              │
                           │ready_for_configuration(→参数计算模块)      │
                           ↓                                           │
                    [Turbo_Encoder_Top] (六状态机)                       │
                    bit→│线性编码→等栅格终止→交织编码→等栅格终止         │
                        │d0(系统位) + d1_d2(校验位交替)                  │
                        │ready_for_code_block(→上游CRC分段模块)          │
                        ↓                                              │
                    [TX_Rate_Matcher] (双页循环缓冲区)                    │
                    d0/d1_d2→│子块交织→bit收集→k0起始bit选择→bit_out    │
                             │ready_for_config(→上游Turbo编码器)         │
                             │ready_for_code_block(→外部)               │
                             ↓                                          │
                    bit_out, bit_out_valid                             │
                    ready_for_data (= CRC模块的ready_for_input)         │
                    ┌──────────────────────────────────────────┐       │
                    │   166-bit 参数总线贯穿全部4个子模块         │       │
                    │   {C, E_idx, K, K+4, Kw, E+, E-, k0,    │       │
                    │    R, full_size, filler, TB_size}        │       │
                    └──────────────────────────────────────────┘       │
```

**关键设计：166-bit参数总线**

这根总线承载了Turbo编码和速率匹配需要的全部参数。它的流动路径：
1. Parameter_Computation 查表产生初始值（组合逻辑，立即）
2. 经过 98 周期延迟后输出给 CRC 分段模块
3. CRC 分段模块在 Wait_for_Ready_Flag 状态将参数广播给 Turbo 和 Rate_Matcher
4. Turbo 模块再将参数延迟 8 周期后传给 Rate_Matcher（因为 Turbo 内部有延迟）

**反压链路（自下而上）：**
```
Rate_Matcher 忙 → ready_for_code_block=0
  → Turbo_Encoder_Top 的 ready_for_code_block=0（delay 3周期）
  → CRC 模块的 ready_for_output=0 → ready_for_input=0
  → Channel_Encoder 的 ready_for_data=0
  → 顶层 ready_for_input=0 → MAC_FIFO 暂停
```

---

#### 6a-1. PXSCH_Parameter_Computation — 查表得到所有编码参数

**解决的问题：** 给定 TB_size，需要知道分几个码块(C)、每个码块多大(K)、速率匹配输出多少bit(E)、循环缓冲区从哪开始读(k0)……这些参数在 LTE 协议（TS36.212）里有复杂的计算公式。但实际系统中只用了8种TB_size，所以——**直接用 case 查表**，不用实现那些公式。

**实现方式：**
```verilog
always @(posedge clk_192M) begin
    case(transport_block_size)
        17'd30576 : begin
            number_of_code_blocks_out_temp <= 5'd5;   // C = 5个码块
            code_block_size_out_temp <= 13'd6144;      // K = 6144 bit/码块
            circular_buffer_size_temp <= 15'd18444;    // Kw = 3×(6144+4)
            E_plus_temp <= 16'd12480;                  // 每码块输出12480 bit
            k0_temp <= 16'd384;                        // 从buffer的第384位开始读
            // ...
        end
        // ... 其他7个TB_size ...
        default : begin  /* 默认用7224的参数 */  end
    endcase
end
```

**全部输出打97拍延迟：**
```verilog
delay_n #(97, 5)  d1(clk_192M, number_of_code_blocks_out_temp, number_of_code_blocks_out);
delay_n #(97, 7)  d2(clk_192M, E_index_temp, E_index);
// ... 共12个参数全部延迟97拍
```

为什么延迟97拍而不是98拍？因为 valid_out 还需要多1拍来生成：
```verilog
delay_n #(98, 1) d13(clk_192M, valid_in, valid_in_delay98);
assign valid_out = ready_for_output && valid_in_delay98;
```

`sync_latch` 控制反压：valid_in 时拉低 ready_for_configuration（告诉上游"我正在算，别给我新参数"），valid_out 时恢复。

**一个值得注意的细节：** 注释掉的旧参数说明这个表经历过调整。比如 `17'd15840` 的 E_plus 从 `16'd10400` 改成了 `16'd7200`，`17'd30576` 的 E_plus 从 `16'd6240` 改成了 `16'd12480`。这些改动直接影响速率匹配的输出bit数，进而影响码率。

**参数含义速查：**

| 参数 | 位宽 | TB=30576时的值 | 含义 |
|------|:--:|:--:|------|
| C | 5 | 5 | 码块数 |
| K | 13 | 6144 | 每码块bit数 |
| K+4 | 13 | 6148 | 加4个tail bits |
| Kw | 15 | 18444 | 循环缓冲区大小 = 3×(K+4) |
| E+ | 16 | 12480 | 前 C-γ-1 个码块的输出bit数 |
| E- | 16 | 12480 | 剩余码块的输出bit数（此处E+=E-，无打孔） |
| k0 | 16 | 384 | 循环缓冲区读取起始位置（RV=0时 = R×32÷?） |
| R | 16 | 193 | 子块交织器行数 = ceil((K+4)/32) |

---

### 💬 讨论：参数查表的两个核心问题

#### Q1: 查表是组合逻辑瞬间出结果，为什么还要延迟 97/98 周期？

这是整个模块最容易让人困惑的地方。case 语句是组合逻辑，参数在同一个时钟周期就算出来了，为什么还要等 98 个周期才给下游？

**原因1：sync_latch 需要一个"忙窗口"来防止参数被覆盖**

这是最主要的功能原因。看这段代码：

```verilog
sync_latch ready_for_PXSCH_Parameter(
    .set(valid_in),      // trigger来了 → 拉高 not_ready
    .clear(valid_out),   // 98周期后 → 拉低 not_ready
    .out(not_ready_for_configuration)
);
assign ready_for_configuration = ~not_ready_for_configuration;
```

sync_latch 是一个 SR 锁存器：
- `valid_in` 上升沿 → `not_ready_for_configuration` 锁存为 1 → `ready_for_configuration` = 0
- 98 周期后 `valid_out` 上升沿 → `not_ready_for_configuration` 清零 → `ready_for_configuration` = 1

在这 98 个周期里，`ready_for_configuration` 一直是 0，这告诉上游："我正在忙，别给我新的 trigger"。

**为什么需要这个忙窗口？** 因为参数从查表到送达下游之间，166-bit 的参数总线正排着队经过 97 级移位寄存器。如果这时候来了第二个 trigger，新的 case 值会立刻覆盖 `_temp` 寄存器，但旧的参数还在移位寄存器里没出来，导致下游收到的参数是"新老混搭"的错乱状态。

用一个比喻：你在餐厅点了菜，厨师立刻知道要做什么（查表），但食材要经过 97 道工序（移位寄存器）才能送到餐桌。如果 97 道工序走完之前你又改了菜单，送出去的菜就对不上了。sync_latch 就是在这 97 道工序期间挂上"暂停接单"的牌子。

**原因2：166-bit 参数总线需要寄存器化来满足时序**

这句话需要展开解释，因为它涉及 FPGA 时序收敛的核心概念。

**先理解基本问题：245.76MHz 意味着什么**

时钟周期 = 1 / 245.76MHz ≈ **4.07ns**。在这个时间窗口内，信号必须完成：
1. 从源寄存器（Launch Register）弹出
2. 穿过所有组合逻辑（LUT、查找表级联）
3. 走过芯片上的物理连线（Routing Delay）
4. 在目的寄存器（Capture Register）的建立时间（Setup Time，约 0.1~0.2ns）之前稳定下来

```
  源寄存器       组合逻辑 + 走线延迟        目的寄存器
  ┌──────┐    ┌─────────────────────┐    ┌──────┐
  │  Q   │───→│  case语句 + 166根线   │───→│  D   │
  └──────┘    │  跨芯片物理走线       │    └──────┘
              └─────────────────────┘
              ↑_____ 必须 < 4.07ns ____↑
```

如果组合逻辑层数太深，或者 166 根线要走很远的物理距离，总延迟超过 4.07ns，目的寄存器采样到的就是稳定之前的不确定值——这就是**时序违例**。

**没有 delay_n 时的信号路径：**

```
Parameter_Computation 模块内:
  _temp 寄存器 → case 语句(组合逻辑) → 模块端口
                                         ↓
                                    166根独立的物理走线
                                    跨过芯片到下游模块
                                         ↓
CRC_Add_and_CB_Segmentation:
  模块端口 → 内部组合逻辑使用 → 目的寄存器
```

整个路径中只有**头尾各一个寄存器**，中间全是组合逻辑和物理走线。166 bit 宽的总线意味着 166 根物理走线要同时到达，最慢的那一根决定了整体速度。如果下游模块距离比较远（比如被工具布局到了芯片另一侧），走线延迟轻松超过 2-3ns，留给组合逻辑的时间就不够了。

**有了 delay_n #(97, W) 之后：**

```
_temp 寄存器 → [case] → delay_n[0] → delay_n[1] → delay_n[2] → ... → delay_n[96] → 下游
              ↑____↑   ↑_________↑   ↑_________↑               ↑__________↑
              第1段     第2段         第3段                      第97段
              每段: 寄存器 → 极短走线 → 寄存器
```

关键变化：**97 级移位寄存器把一条 166-bit 宽的长路径切成了 97 段短路径**。

每一段的结构是：
```
  delay_n[i]  →  走线(同级寄存器之间)  →  delay_n[i+1]
  寄存器Q端       距离非常短              寄存器D端
```

每个 delay_n 的实例就是 166 个并行的 D 触发器。从一个 delay_n 的 Q 端到下一个 delay_n 的 D 端，中间**没有任何组合逻辑**，只有纯走线延迟。Vivado 会把相邻的 delay_n 寄存器布局得非常近，走线延迟极短（可能不到 0.5ns）。每段只需满足：走线延迟 < 4.07ns——这几乎是自动满足的。

**工具可以"自由流水线化"是什么意思？**

如果 case 语句本身的组合逻辑也很深（虽然这里是查表不太深），工具可以把 case 的 LUT 层级拆分到 delay_n 的前几级：

```
_temp → [case第1层LUT] → delay_n[0] → [case第2层LUT] → delay_n[1] → ... → 下游
```

这就是"寄存器重定时（Retiming）"——Vivado 自动把组合逻辑往前或往后推，分配到各段寄存器之间，让每段的组合逻辑+走线都小于 4.07ns。

**为什么恰恰是 97 级？**

97 级并不是精确计算出来的。只要"足够多"，保证：
1. 98 周期的 sync_latch 忙窗口足够长，挡住后续 trigger
2. 下游各模块有时间完成初始化
3. 97 级移位寄存器让工具在芯片上有足够的物理空间来"铺开"这 166 根走线

**一个类比：**

没有 delay_n 就像你要把 166 箱货从城东仓库直接搬到城西商店，一趟必须搬完，中途不能放下——最重的那箱决定了你能不能按时到。

有了 delay_n 就像在城东和城西之间设了 97 个中转站，每个中转站你都可以放下货休息（寄存器锁存），下一段只需要搬到相邻的中转站。每段路程都很短，轻松按时到达。货物经过 97 次接力，虽然总时间更长，但每一段都稳稳当当不会超时。

**原因3：参数延迟要和下游模块的就绪时间对齐**

参数最终要被 CRC_Add_and_CB_Segmentation 使用。在那个模块里，参数到达（`configuration_valid_in`）后，状态机从 IDLE → Wait_for_Ready_Flag → Write_Input_Data。但 Write_Input_Data 需要等 Turbo_Encoder_Top 的 `ready_for_output`。98 周期的延迟给了 Turbo 编码器足够的初始化时间。

**为什么恰恰是 97/98？** 源文件注释里写了一句 `时延为98，34（第1个模块）+63（第2个模块）+1（最后一个脉冲）`，但这个注释看起来是从别的模块复制来的，不太可信。更可能的情况是：这是实际调试中确定的值——够用、整数、好记。不是某个公式算出来的精确值。

**一个常见的误解：delay_n 不是"原地等 97 拍"**

很多人第一次看到 `delay_n #(97, W)` 会以为："哦，就是让值保持不变，等 97 个周期再输出"。**不是这样的。**

`delay_n #(97, W)` 的内部是 97 级移位寄存器链：

```
  输入 ──→ [Reg_0] ──→ [Reg_1] ──→ [Reg_2] ──→ ... ──→ [Reg_96] ──→ 输出
            D  Q        D  Q        D  Q                  D  Q
            ↑            ↑            ↑                     ↑
          clk          clk          clk                   clk
```

每个时钟上升沿，值向前移动一级：
- Reg_0 锁存当前输入
- Reg_1 锁存上一拍 Reg_0 的值
- Reg_2 锁存上一拍 Reg_1 的值
- ...
- 97 个周期后，值从 Reg_96 送出

这是个**流水线传送带**，不是"保持 97 个周期"。打个比方：97 个人排成一队传递包裹，每个人接过来、转身、递给下一个人——包裹每周期前进一个人，97 个周期后到达队尾送出。

**以 TB=30576 为例的时间线：**

```
周期 0:  trigger 到达, valid_in=1
         case语句瞬间算出 _temp = {C=5, K=6144, E=12480, ...}
         _temp 被锁存进 Reg_0
         输出仍是旧值/0（Reg_96 还没收到）

周期 1:  Reg_0(30576的参数) → Reg_1
         Reg_0 锁存新的输入（sync_latch关门前，还是同一个值）

周期 2:  Reg_1 → Reg_2
...

周期 96: 30576的参数到达 Reg_96

周期 97: Reg_96 → 输出端口
         参数终于出现在输出端！

周期 98: valid_in 也经过 delay_n #(98,1) 完成
         valid_in_delay98=1 → valid_out=1
         下游同时收到: "参数有效 + TB=30576 + C=5 + K=6144 + ..."
```

**这个延迟的本质：参数和 MAC_FIFO 数据各走各的路，但要在 CRC 模块门口同时到达**

这就是所谓的"延迟匹配"。从 trigger 发出的那一刻起：

```
   trigger ──→ Parameter_Computation (98周期固定延迟, 查表+排队)
            │                           ↓
            │                   参数到达 CRC 模块门口
            │
            └──→ MAC_FIFO (可变延迟, 发帧头+发数据)
                                         ↓
                                 数据 bit 到达 CRC 模块门口
```

参数走的是"快速通道"——瞬间查表，然后排队 97 周期。数据走的是"慢速通道"——MAC_FIFO 要先发 32 周期帧头，才开始一个字节一个字节地发 payload。97 周期后，数据差不多该到了，参数正好送到，不早不晚。

如果用真值表来理解：97 不是计算出来的精确匹配值，而是"总够用"的安全裕量。只要参数在数据之前到达 CRC 就行——参数先到了可以等（ready_for_output 握手机制），但数据到了参数还没到就出错了。所以宁可让参数早点出门排队，也不能让它迟到。

---

#### Q2: 这些参数到底是怎么来的？——以 TB=30576 为例完整推导

查表里的 12 个参数不是随便填的，每一个都对应 LTE 协议 TS36.212 里的公式。下面以最大的 TB=30576 为例，从源头一步步推导。

**第零步：TB_size 是怎么确定的？**

在 Parameter_Computation 之前，TB_size 已经由 MCS_to_TBS 模块确定了：

```
MCS_control (VIO手动开关, 0~3)
    │
    ↓
MCS_to_TBS 查表:
    MCS=0: subframe0→7224,   其他→10296
    MCS=1: subframe0→8760,   其他→15840
    MCS=2: subframe0→15840,  其他→29296
    MCS=3: subframe0→8760,   其他→30576   ← 最大配置
    │
    ↓
TB_size = 30576 → 送入 Parameter_Computation
```

同时，调制方式也由 MCS_control 决定：
- MCS=0/1/2 → QPSK（每个符号 2 bit）
- MCS=3 → 16QAM（每个符号 4 bit）

**第一步：码块分割 (TS36.212 5.1.2)**

```
TB = 30576 bit
B = TB + CRC24A = 30576 + 24 = 30600 bit     (加上传输块CRC)

最大码块大小 Kmax = 6144

是否需要分割？B = 30600 > 6144 → 需要分割

码块数 C = ceil(B / (6144-24))               (减24是因为每个码块也要加CRC)
         = ceil(30600 / 6120)
         = ceil(5.0)
         = 5

总bit数(含所有CRC) B' = B + C×24 = 30600 + 120 = 30720

每码块bit数 K = 能使 C×K ≥ B' 的最小允许值
  C×K ≥ 30720 → K ≥ 6144
  查 TS36.212 Table 5.1.3-3，K=6144 恰好满足: 5×6144 = 30720 ≥ 30720 ✓
  
填充bit数 F = C×K - B' = 30720 - 30720 = 0   (刚好不需要填充)
```

**第二步：Turbo 编码后的大小 (TS36.212 5.1.3)**

```
每个码块 K=6144 bit 送入 Turbo 编码器
Turbo 编码后加 4 个栅格终止bit (tail bits)
每码块系统位: K+4 = 6148 (number_of_systematic_bits)

Turbo 码率 = 1/3: 1路系统位(d0) + 2路校验位(d1+d2)
循环缓冲区大小 Kw = 3 × (K+4) = 3 × 6148 = 18444 (circular_buffer_size)
```

**第三步：子块交织器参数 (TS36.212 5.1.4.1.1)**

```
子块交织器: 32列固定

行数 R = ceil((K+4) / 32) = ceil(6148/32) = ceil(192.125) = 193
       (number_of_interleaver_rows)

满交织器大小 = R × 32 = 193 × 32 = 6176 (full_interleaver_size)

填充bit数 = 6176 - 6148 = 28 (number_of_interleaver_filler_bits)
  → 每路(d0/d1/d2)填入28个<NULL>，凑满193×32矩阵
```

**第四步：速率匹配输出大小 (TS36.212 5.1.4.1.2)**

```
可用RE数: RE_NUMBER1 = 15600 (非子帧0)
调制阶数: 16QAM → 每符号4bit → mod_order = 4
总编码bit数: G = 15600 × 4 = 62400

每码块输出bit: E = G / C = 62400 / 5 = 12480 (E_plus = E_minus)

说明此处 E_plus = E_minus，意味着没有打孔，所有码块输出相同bit数。
对应协议中 γ = C 的情况（所有码块都用E+）。
```

**第五步：速率匹配起始位置 k0 (TS36.212 5.1.4.1.2)**

```
冗余版本 RV = 0 (固定)
k0 = R × (2 × ceil(N_cb/(8×R)) × RV + 2)

N_cb = Kw = 18444 (对于下行，N_cb = Kw)

如果按公式严格计算: k0 = 193 × 2 = 386

但表中写的是 k0 = 384 = 192 × 2

这个微小差异说明实际计算中可能用了 R = floor(Kπ/32) = 192,
而非 ceil(Kπ/32) = 193。

或者 R_subblock 的计算有调整。
```

**第六步：E_index（用于区分E+和E-的码块数）**

```
E_index 表示有多少个码块使用 E-（较小的输出大小）
当 E_plus = E_minus 时，E_index = C-1 = 4

对于 TB=30576: E_index = 4
```

---

**验证：用同样的方法验证 TB=10296**

```
TB = 10296, subframe≠0 → RE=15600, QPSK → mod_order=2

B = 10296 + 24 = 10320
C = ceil(10320/6120) = ceil(1.687) = 2
B' = 10320 + 2×24 = 10368
K: 2×K ≥ 10368 → K ≥ 5184 → 查表得 K=5184 ✓
F = 2×5184 - 10368 = 0

K+4 = 5188 ✓
Kw = 3 × 5188 = 15564 ✓
R = ceil(5188/32) = ceil(162.125) = 163 ✓
full_size = 163 × 32 = 5216 ✓
filler = 5216 - 5188 = 28 ✓

G = 15600 × 2 = 31200
E = 31200 / 2 = 15600 ✓
```

**验证 TB=15840（子帧0）：**

```
TB = 15840, subframe=0 → RE=10800, QPSK → mod_order=2

B = 15840 + 24 = 15864
C = ceil(15864/6120) = ceil(2.592) = 3
B' = 15864 + 3×24 = 15936
K: 3×K ≥ 15936 → K ≥ 5312 → 查表得 K=5312 ✓
F = 3×5312 - 15936 = 0

K+4 = 5316 ✓
Kw = 3 × 5316 = 15948 ✓
R = ceil(5316/32) = ceil(166.125) = 167 ✓
full_size = 167 × 32 = 5344 ✓
filler = 5344 - 5316 = 28 ✓

G = 10800 × 2 = 21600
E = 21600 / 3 = 7200 ✓
```

---

**汇总：12个参数的来源公式**

| 参数 | 来源 | 公式 |
|------|------|------|
| **C** | TS36.212 5.1.2 | ceil((TB+24) / (6144-24)) |
| **K** | TS36.212 Table 5.1.3-3 | 使 C×K ≥ TB+24+C×24 的最小K |
| **K+4** | TS36.212 5.1.3 | K + 4 (tail bits) |
| **Kw** | TS36.212 5.1.4 | 3 × (K+4) |
| **R** | TS36.212 5.1.4.1.1 | ceil((K+4) / 32) |
| **full_interleaver_size** | TS36.212 5.1.4.1.1 | R × 32 |
| **filler_bits** | TS36.212 5.1.4.1.1 | R×32 - (K+4) |
| **E_plus** | TS36.212 5.1.4.1.2 | floor(G / C), 其中 G = RE_number × mod_order |
| **E_minus** | TS36.212 5.1.4.1.2 | ceil(G / C) |
| **E_index** | TS36.212 5.1.4.1.2 | C - γ - 1 (使用E-的码块数) |
| **k0** | TS36.212 5.1.4.1.2 | R × (2 × ceil(N_cb/(8×R)) × RV + 2) |
| **TB_size** | MCS_to_TBS | 由MCS档位+子帧号查表 |

**为什么用查表而不是实现这些公式？**

因为这些公式里有 `ceil`、查 Table 5.1.3-3（188个离散K值）、条件分支，用硬件实现需要好几个周期且非常占资源。而系统只有 8 种 TB_size，直接写死 8×12=96 个常数，一个 case 语句瞬间完成，省时省力省面积。

---

#### Q3: 166-bit 参数总线到底是什么？

这个"166-bit 参数总线"在讨论中反复提到，但一直没展开讲。它就是把 12 个参数拼成一根 166 位宽的总线，让它们在编码链的四个子模块之间同步传递。

**为什么是 166？**

在 Turbo_Encoder_Top.v 的第299行有明确注释：

```verilog
delay_n #(
    .N(8),    // 延迟8个周期
    .Width(166) // 信号位宽 = 5+7+13+13+15+16+16+16+16+16+16+17
)
```

把 12 个参数的位宽加起来：

```
number_of_code_blocks      5 bit    (C: 码块数, 最大31够用)
E_index                    7 bit    (r值: 区分E+/E-的码块数)
code_block_size           13 bit    (K: 每码块bit数, 最大6144=13bit)
number_of_systematic_bits 13 bit    (K+4: 最大6148=13bit)
circular_buffer_size      15 bit    (Kw: 最大18444=15bit)
E_plus                    16 bit    (速率匹配输出bit数, 最大12480)
E_minus                   16 bit    (同上)
k0                        16 bit    (循环缓冲区起始位置)
number_of_interleaver_rows 16 bit   (R: 子块交织器行数, 最大193)
full_interleaver_size     16 bit    (R×32: 最大6176)
number_of_interleaver_filler_bits 16 bit (填充bit数, 最大28)
transport_block_size      17 bit    (TB_size: 最大30576=15bit, 留17bit裕量)
────────────────────────────────
总和                      166 bit
```

**这根总线的流动路径：**

```
PXSCH_Parameter_Computation (查表产生)
  │  12个参数拼成166-bit, delay 97周期
  ↓
CRC_Add_and_CB_Segmentation (透传, 加CRC期间参数不变)
  │  Wait_for_Ready_Flag时绑到输出端口
  │  {C,E_idx,K,K+4,Kw,E+,E-,k0,R,full,filler,TB}
  ↓
Turbo_Encoder_Top (透传, delay 8周期)
  │  delay_n #(8,166): 等Turbo内部状态机完成
  │  172行注释: "时延8个周期的原因是参数应当只要比ready for code block晚就行"
  ↓
TX_Rate_Matcher (最终消费)
  │  根据这些参数执行子块交织+bit收集+bit选择
  │  FSM用C判断有几个码块要处理
  │  Linear_Address_Generator用k0和Kw生成读地址
  │  Interleaved_Address_Generator用R和filler生成写地址
```

**为什么要把它们打包成一根总线？**

FPGA 里没有"函数调用"和"参数传递"。如果每个参数单独用一根 wire 连接四个子模块，需要写 12 根独立的 delay 延迟线，代码冗长且容易漏连。打包成一根总线后，只需要一个 `delay_n #(N, 166)` 实例，12 个参数就全部同步延迟了。

```verilog
// 没有总线时的写法（需要12个delay_n，共12×N个寄存器）
delay_n #(N,5)  d1 (clk, C_in, C_out);
delay_n #(N,7)  d2 (clk, E_idx_in, E_idx_out);
delay_n #(N,13) d3 (clk, K_in, K_out);
// ... 写12遍，每个参数都要声明wire

// 有总线后的写法（1个delay_n，内部12×N个寄存器自动管好）
delay_n #(N, 166) d_all (clk, {C,E_idx,K,K4,Kw,E+,E-,k0,R,full,filler,TB},
                               {C,E_idx,K,K4,Kw,E+,E-,k0,R,full,filler,TB});
```

**总线在代码中的物理形态：**

在 PXSCH_Channel_Encoder.v 中，四个子模块之间是分开引出的 wire（为了可读性），但每个子模块接收到的参数完全一样——都是从 Parameter_Computation 广播出来的同一组值，然后打包延迟：

```verilog
// PXSCH_Channel_Encoder.v 的实例化:
// Parameter_Computation → CRC (12根独立wire, 内容完全等同于166-bit总线)
CRC_Add_and_CB_Segmentation (
    .number_of_code_blocks_in(number_of_code_blocks_para),  // 5
    .E_index_in(E_index_para),                              // 7
    .code_block_size_in(code_block_size_para),              // 13
    // ... 其余9个参数 ...
);

// CRC → Turbo (又是12根独立wire)
Turbo_Encoder_Top (
    .number_of_code_blocks_in(number_of_code_blocks_crc),   // 5
    .E_index_in(E_index_crc),                               // 7
    // ...
);

// Turbo → Rate_Matcher (打包成166-bit总线, delay 8)
delay_n #(8, 166) delay_Turbo_Encoder_Top_configuration_parameter(
    .In({number_of_code_blocks_in, E_index_in, ...}),        // 拼接
    .Out({number_of_code_blocks_out, E_index_out, ...})      // 拆分
);
```

**每个参数在 TB=30576 时的值：**

| 参数 | 位宽 | TB=30576时的值 | 二进制 | 用途 |
|------|:--:|:--:|------|------|
| C | 5 | 5 | `00101` | Turbo和RM知道要处理几个码块 |
| E_index | 7 | 4 | `0000100` | E+/E-的个数分界 |
| K | 13 | 6144 | `1100000000000` | CRC分段基准、Turbo编码长度 |
| K+4 | 13 | 6148 | `1100000000100` | RM子块交织输入大小 |
| Kw | 15 | 18444 | `100100000001100` | 循环缓冲区总大小 |
| E+ | 16 | 12480 | `0011000011000000` | 每码块RM输出bit数 |
| E- | 16 | 12480 | `0011000011000000` | （此处E+=E-相同） |
| k0 | 16 | 384 | `0000000110000000` | 循环缓冲区读起始位置 |
| R | 16 | 193 | `0000000011000001` | 子块交织器行数 |
| full_size | 16 | 6176 | `0001100000100000` | 满交织器大小(R×32) |
| filler | 16 | 28 | `0000000000011100` | 填充<NULL>的个数 |
| TB_size | 17 | 30576 | `0111011101110000` | 传给RM用于末尾对齐 |

166 bit 就是这 12 个参数的二进制值按顺序拼在一起。当 Parameter_Computation 的 98 周期延迟结束后，这 166 个 bit 就像一列火车一样，从 CRC 模块开到 Turbo 模块，再开到 Rate_Matcher 模块，每个模块取自己需要的字段就行了。

---

#### 6a-2. CRC_Add_and_CB_Segmentation — CRC添加 + 码块分段

**解决的问题：** 
1. 给整个传输块加 CRC-24A 校验（接收端用来判断整个TB对不对）
2. 如果TB太大（C>1），切成多个码块，每个码块再加 CRC-24B
3. CRC校验bit要插在数据末尾（不是开头也不是独立发送）

**五状态机流转：**

```
  trigger到达
      ↓
  ┌─────────┐   configuration_valid_in   ┌──────────────────┐
  │  IDLE   │ ─────────────────────────→ │ Wait_for_Ready_  │
  │  (空闲)  │ ←───────────────────────── │ Flag (等待下游)   │
  └─────────┘   所有码块处理完毕           └────────┬─────────┘
      ↑                                            │ ready_for_output=1
      │                                     ┌──────↓──────────┐
      │                                     │ Write_Input_Data │
      │                                     │ (逐bit接收TB数据) │
      │                                     └──┬──────┬───────┘
      │                          TB数据全部收完  │      │ C>1且码块只剩24bit
      │                              ┌──────────↓──┐ ┌─↓───────────┐
      │                              │ Write_TB_CRC│ │ Write_CB_CRC │
      │                              │ (输出24bit  │←│ (输出24bit   │
      │                              │  TB CRC)    │ │  码块CRC)    │
      │                              └────┬───────┘ └──┬───────────┘
      │                   不需要分段→IDLE │   还有码块→Wait │
      └─────────────────────────────────┘                │
```

**状态机代码实现（三段式中的第1段——状态转移）：**

```verilog
always @(posedge clk_192M) begin
    case(Current_state)
        IDLE : Current_state <= configuration_valid_in ? Wait_for_Ready_Flag : Current_state;
        Wait_for_Ready_Flag : Current_state <= ready_for_output ? Write_Input_Data : Current_state;
        Write_Input_Data : begin
            if(remaining_bits_in_transport_block_minus1 == 17'd0)
                Current_state <= Write_TB_CRC;              // TB收完→发TB CRC
            else if(code_block_segmentation & (remaining_bits_in_code_block_minus1 == 5'd24))
                Current_state <= Write_CB_CRC;              // 码块收完→发CB CRC
        end
        Write_CB_CRC : Current_state <= (remaining_CRC_bits_minus1 == 5'd0) ?
            ((remaining_code_blocks_state_machine_minus1 > 5'd0) ? Wait_for_Ready_Flag : IDLE) : Current_state;
        Write_TB_CRC : Current_state <= (remaining_CRC_bits_minus1 == 5'd0) ?
            (code_block_segmentation_state_machine ? Write_CB_CRC : IDLE) : Current_state;
    endcase
end
```

**关键设计：`_minus1` 寄存器**

状态机里大量使用 `xxx_minus1` 寄存器——这是当前值减1。为什么？

```verilog
// 在状态2(Write_Input_Data)的判断：
if(remaining_bits_in_transport_block_minus1 == 17'd0)  // 意味着remaining_bits_in_transport_block == 1
```

因为状态转移是组合逻辑（或本周期判断），需要提前1周期知道"下一拍是不是最后一拍"——这一拍数据来了之后计数器才减1，但状态跳转必须同一拍发生。`_minus1` 寄存器解决了这个"提前量"问题。

**两级CRC串联的流水线：**

```
  bit_in ──→ [TB_CRC_Calculation] ──→ [CB_CRC_Calculation] ──→ bit_out
              │ 多项式: gCRC24A      │ 多项式: gCRC24B
              │ 实时计算，输出=输入   │ 实时计算，输出=输入
              │ (数据直通，同时算CRC) │ (数据直通，同时算CRC)
```

两个 `CRC24_Adder` 模块串在一起。数据直通（bit_in 延迟1拍后直接到 bit_out），CRC 值在后台实时更新。当 trigger_CRC_output 脉冲到来时，用一个24周期的 strobe 把预计算好的 CRC 值逐bit插入数据流。

**CRC24_Adder 内部实现（关键！）：**

```verilog
// CRC计算模块（组合逻辑+寄存器）
CRC_Calculation calculate_tb_crc(
    .bit_in(bit_in),           // 逐bit输入
    .bit_in_valid(bit_in_valid),
    .polynomial(polynomial),   // 生成多项式
    .CRC(continuous_CRC)       // 实时更新的24-bit CRC值
);

// CRC输出控制：生成24周期strobe
PULSE_TO_STROBE_U16 #(5) permit_crc_out(
    .start_pulse(trigger_CRC_output),  // 启动脉冲
    .N(24),                            // 持续24周期
    .strobe(strobe),                   // 24周期高电平
    .index(index)                      // 0→23计数器
);

// 输出选择：strobe期间输出CRC bit，否则数据直通
assign bit_out = strobe_delay1 ? continuous_CRC_reg[index_delay1] : bit_in_delay1;
```

这实现了一个巧妙的功能：正常时数据直通，需要发CRC时"截断"数据流，插入24个CRC bit，发完继续正常数据。

---

#### 6a-3. Turbo_Encoder_Top — 1/3码率Turbo编码（最复杂的子模块）

**解决的问题：** 把每个码块的 K 个 bit，通过 Turbo 编码变成约 3K 个 bit（系统位+两个校验位），大幅增强纠错能力。

**Turbo编码的基本原理（人话版）：**

Turbo编码 = 两个相同的RSC编码器 + 一个交织器。

```
  bit_in ──────┬──────────────→ d0 (系统位原样输出)
               │
               ├──→ [RSC编码器1] ──→ d1 (第一个校验位)
               │
               └──→ [交织器打乱] → [RSC编码器2] ──→ d2 (第二个校验位)
```

核心思想：
- 系统位 d0 = 原始数据（不编码，直接传）
- 校验位 d1 = 原始顺序编码的结果
- 校验位 d2 = 打乱顺序后再编码的结果（同一个编码器，不同输入顺序）

为什么要打乱？因为如果信道在某处深衰落，打乱后错误被分散，接收端迭代译码更容易纠错。

**六状态机设计：**

```
         new_code_block
  IDLE ─────────────────→ Encode_Linear_Codeblock (线性编码 K bit)
                              │ bit数够了
                              ↓
                           Wait_for_d0_d1_Termination_Bits (等3个栅格终止bit)
                              │ counter>=3
                              ↓
                           Encode_Interleaved_Codeblock (交织后编码 K bit)
                              │ bit数够了
                              ↓
                           Wait_for_d2_Termination_Bits (等3个栅格终止bit)
                              │ counter>=3
                              ↓
                           Wait_for_Termination_Bit_Output (等8拍输出完)
                              │ counter>=8
                              ↓
                           IDLE
```

**为什么一个码块要编码两遍？** 因为只有一个物理 Turbo_Encoder。第一遍：按原始顺序输入，产生 d0 + d1。第二遍：从交织器读出来按打乱的顺序输入，产生 d2。"两个编码器"实际是同一个硬件复用。

**7个子模块的协作流程：**

```
状态1 (线性编码期间):
  bit_in ──→ [Interleaver_Write_Stream_Generator] ──→ Interleaver_Buffer (写入)
  bit_in ──→ (延迟7周期) ──→ [Turbo_Encoder] ──→ d0(系统位), z(线性校验)
                               ↑ num_encoded_bits 计数
                               │ 当 count≥K 时触发状态切换
                               
状态2 (等栅格终止):
  Turbo_Encoder 继续跑3个周期 → 产生 d0+d1 的栅格终止bit
  [Store_Termination_Bits] 开始存储这3+3=6个终止bit
  同时 Interleaver 的 read_enable 拉高，准备读
  
状态3 (交织编码期间):
  [Interleaver_Read_Stream_Generator] 生成 0→K-1 顺序读地址
  Interleaver_Buffer 读出打乱的数据
  读出的bit → [Turbo_Encoder] (同一物理编码器) → x'(交织系统位), z'(交织校验位)
  
状态4 (等栅格终止):
  再跑3个周期 → 产生 d2 的栅格终止bit
  Store_Termination_Bits 存完剩余6个终止bit（共12个）
  
状态5 (输出栅格终止):
  [Map_Termination_Bits_To_Streams] 把12个终止bit插入 d0/d1_d2 输出流
  持续8周期
```

**延迟匹配设计（这很关键！）：**

数据到达 Turbo_Encoder 需要等交织器写完才能读。延迟匹配：
```
bit_in → (delay 7周期) → Turbo_Encoder.bit_in
         = 交织器写入3周期 + 等交织器地址流水线 + 读出4周期
```

配置参数 delay 8周期给下游 Rate_Matcher：
```
new_code_block → (delay 8周期) → configuration_out_valid
               = 状态机1周期 + 交织器7周期
```

**Turbo_Encoder 核心实现（最底层编码器）：**

```verilog
module Turbo_Encoder(
    input bit_in,              // 输入bit
    input bit_in_valid,        // 输入有效
    input [12:0] code_block_size,       // K
    input [12:0] code_block_size_add3,  // K+3（多3个栅格终止bit）
    output x_out,              // 系统位输出
    output z_out,              // 校验位输出
    output output_valid
);

// 核心：Turbo_Encoder_Filter（RSC递归系统卷积编码器，组合逻辑+寄存器）
Turbo_Encoder_Filter exe_turbo_encoding(
    .bit_in((num_encoded_bits < code_block_size) ? bit_in : termination_bit_x_delay1),
    //       ↑ 编码K个数据bit时用输入bit，超过K后用反馈的栅格终止bit
    .input_valid(((num_encoded_bits >= code_block_size) | bit_in_valid) & (code_block_size != 13'd0)),
    //            ↑ 数据有效 || 栅格终止期间（再跑3周期）
    .bit_out(z_out_ahead),               // 校验位（组合逻辑计算）
    .termination_bit_x(termination_bit_x) // 反馈bit（用于栅格终止）
);
```

**num_encoded_bits 计数器：**
```verilog
// 编码了K+3个bit后归零（K个数据 + 3个栅格终止）
num_encoded_bits <= (num_encoded_bits >= code_block_size_add3) ? 13'd0 : (num_encoded_bits + 1'b1);
```

`code_block_size_add3 = K+3`，多出的3个周期用于栅格终止：编码器不再接收外部数据，而是把自己的输出反馈回输入（`termination_bit_x_delay1`），跑3个周期让内部状态归零。

**Store_Termination_Bits — 如何收集12个栅格终止bit：**

Turbo编码结束后，需要输出12个栅格终止bit（d0的3个 + d1的3个 + d2的6个）。但它们是交错着来的，没法直接串行发送。于是用一个 6-cycle 的 Mod_N_Indexer 来"分拣"：

```
bit_in_valid 每来1拍，index 从0→5循环：

index=0: x[k]→d0[0],  z[k]→d1[0]
index=1: x[k+1]→d0[1],  z[k+1]→d2[0]   ← d1和d2开始交替
index=2: x[k+2]→d1[1],  z[k+2]→d2[1]   ← d0没有数据
index=3: x'[k]→d0[2],  z'[k]→d1[2]
index=4: x'[k+1]→d0[3],  z'[k+1]→d2[2]
index=5: x'[k+2]→d1[3],  z'[k+2]→d2[3]
```

三个4-bit数组（array_1=d0的终止bit, array_2=d1, array_3=d2），每个周期根据 index 和 control 信号，用异或翻转对应位置：
```verilog
array_1 <= (bit_1 == array_1[index_1]) ? array_1 : (array_1 ^ (4'd1 << index_1));
// 如果新bit和当前bit不同，翻转该位（等于写入新值）
```

最终 `termination_bits = {array_3, array_2, array_1}` = 12-bit栅格终止序列。

**Map_Termination_Bits_To_Streams — 如何把终止bit插入输出流：**

用一个 8-cycle 的计数器，前4周期替换d0和d1_d2，后4周期只替换d1_d2：

```
cycle 0-3: d0 = termination[0:3],  d1_d2 = termination[4:7]
cycle 4-7: d0 = d0_in(直通),       d1_d2 = termination[8:11]
```

`sync_latch` 确保栅格终止期间 `termination_bits_valid_latch` 保持高电平8个周期。

---

### 💬 讨论：交织器、栅格终止比特、RSC编码器的实现细节

这几个问题深入到 Turbo 编码器的最底层实现，需要对照 TS36.212 的图 5.1.3-2 和硬件源码来理解。

#### Q1: RSC 递归系统卷积编码器是什么？怎么实现的？

**先说什么是卷积编码器**

普通的"无记忆"编码：输入 3 个 bit，按某种规则输出 6 个 bit。每个输出只取决于当前的输入。

卷积编码器不一样：**输出不仅取决于当前输入，还取决于之前若干拍的输入**。它有"记忆"——内部有移位寄存器保存历史状态，像一个持续运转的状态机。

**再说 RSC（递归系统卷积）**

LTE Turbo 码的 RSC 编码器，源码在 `Turbo_Encoder_Filter.v`，只有 **3 个寄存器 + 3 个异或门**：

```verilog
reg bit_delay1;  // D1: 第1级移位寄存器
reg bit_delay2;  // D2: 第2级移位寄存器
reg bit_delay3;  // D3: 第3级移位寄存器
```

对应 TS36.212 图 5.1.3-2 的硬件结构：

```
                        ┌──────────┐
  bit_in ──→ [⊕] ──→ [D1] ──→ [D2] ──→ [D3]
              ↑        │        │        │
              │        │        │        │
              └──[⊕]───┘        │        │
                        ┌───────┘        │
                        │  ┌─────────────┘
                        ↓  ↓
              [⊕] ← ← ← ← ←←
                │
                ↓
         termination_bit_x (= D1 ⊕ D2, 用于栅格终止)
```

**三条输出路径：**

| 输出 | 表达式 | 含义 |
|------|--------|------|
| 系统位 (x) | bit_in 直通 | 原样输出，不经过任何编码 |
| 校验位 (z) | bit_in ⊕ D2 ⊕ D1 | 当前输入 ⊕ 前两个状态 |
| 栅格终止反馈 | D1 ⊕ D2 | 编码结束后用于清空状态 |

**源码和公式的对照：**

```verilog
// 校验位输出（组合逻辑，0延迟）
assign bit_out = bit_in ^ bit_delay2 ^ bit_delay1;
// 即 z(n) = x(n) ⊕ D2(n-1) ⊕ D1(n-1)

// 栅格终止反馈
assign termination_bit_x = bit_delay1 ^ bit_delay2;
// 即 feedback = D1 ⊕ D2

// 状态更新（每个时钟沿）
bit_delay1 <= bit_in ^ bit_delay2 ^ bit_delay3;  // D1 = 输入 ⊕ D2 ⊕ D3 (递归反馈!)
bit_delay2 <= bit_delay1;                         // D2 = D1(上一拍)
bit_delay3 <= bit_delay2;                         // D3 = D2(上一拍)
```

**"递归"体现在哪里？**

普通的卷积编码器：D1 的输入就是 bit_in，没有反馈。
RSC 的"递归"：**D1 的输入 = bit_in ⊕ D2 ⊕ D3**，即输出又喂回输入。

```verilog
bit_delay1 <= bit_in ^ bit_delay2 ^ bit_delay3;
//            ↑       ↑反馈路径：D2和D3又参与下一次计算
```

这个反馈回路是 Turbo 码纠错能力的关键——它让编码器像一个 IIR 滤波器，有无限冲激响应，使得每个输入 bit 的影响延续到后续所有的输出中。

**"系统"体现在哪里？**

"系统码"意味着**原始数据 bit 作为输出的一部分原样发送**（不编码）。在 Turbo_Encoder 模块中：

```verilog
// Turbo_Encoder.v 第92行
x_out_reg <= (num_encoded_bits < code_block_size) ? bit_in : termination_bit_x_delay1;
```

正常编码期间，系统位输出 x = bit_in（直通）。栅格终止期间，x = 反馈bit。

**完整的数据流：**

```
bit_in ──→ x_out (系统位, 直通, 1周期延迟)
        └─→ Turbo_Encoder_Filter ──→ z_out (校验位, 1周期延迟)
                │
                └─→ termination_bit_x (栅格终止反馈)
```

编码器输出两路：系统位 d0 = x_out，校验位 d1_d2 = z_out。Turbo 码率 1/3 就是这样来的——1 个输入 bit 产生 3 个输出 bit（d0 + d1 + d2，其中 d1 和 d2 是同一个物理编码器跑两次产生的）。

---

#### Q2: 栅格终止比特是什么？

**问题：编码器有状态（D1/D2/D3），编码完一个码块后怎么清空？**

RSC 编码器有 3 个寄存器，编码完 K 个 bit 后，这 3 个寄存器里还残留着历史的"记忆"。如果不处理：
- 接收端用同样的编码器开始解码时，初始状态对不上
- 最后一个码块的尾部 bit 会更容易出错

**解决方案：栅格终止（Trellis Termination）**

编码完 K 个数据 bit 后，不再输入新数据，而是把**编码器自己的反馈作为输入**，再跑 3 个周期。这样 3 个寄存器会自然归零。

```verilog
// Turbo_Encoder.v 第77行：编码bit的选择
.bit_in((num_encoded_bits < code_block_size) ? bit_in : termination_bit_x_delay1)
//      ↑ 正常编码期间: 用外部输入bit        ↑ 栅格终止期间: 用反馈bit

// 第79行：input_valid在栅格终止期间继续为高
.input_valid(((num_encoded_bits >= code_block_size) | bit_in_valid) & (code_block_size != 13'd0))
//           ↑ 栅格终止期间继续编码3个周期
```

**栅格终止期间发生了什么？**

```
正常编码最后1bit (周期K-1):
  bit_in = 数据bit, D1/D2/D3 = 某个状态值
  
栅格终止周期1 (周期K):
  bit_in = termination_bit_x = D1 ⊕ D2  ← 反馈代替输入
  编码器继续运转, 输出1个终止bit (属于d0和d1)

栅格终止周期2 (周期K+1):
  bit_in = 新的 termination_bit_x
  编码器继续, 输出第2个终止bit

栅格终止周期3 (周期K+2):
  bit_in = 新的 termination_bit_x
  编码器继续, 输出第3个终止bit
  此时 D1/D2/D3 全部归零 ✓
```

**为什么需要收集 12 个终止bit？**

Store_Termination_Bits 用 6-cycle Mod_N_Indexer 收集 12 个终止bit：

```
6个index周期，Turbo编码器输出一对 (x, z):

index=0: x[K],   z[K]    → d0[0], d1[0]
index=1: x[K+1], z[K+1]  → d0[1], d2[0]  (注意!d1这拍没数据,d2有)
index=2: x[K+2], z[K+2]  → d1[1], d2[1]  (d0这拍没数据)
index=3: x'[K],  z'[K]   → d0[2], d1[2]  (交织编码的终止bit)
index=4: x'[K+1],z'[K+1] → d0[3], d2[2]
index=5: x'[K+2],z'[K+2] → d1[3], d2[3]
```

最终：d0(4bit) + d1(4bit) + d2(4bit) = 12bit 栅格终止序列。注意不是简单的"每路分3个终止bit"——x和z的输出交错分配给d0/d1/d2，需要 Store_Termination_Bits 这个硬件来做"分拣"。

**一个的例子：**

```
 │ 周期 │ feedback (=上一拍D1⊕D2) │ D2 (当前) │ D3 (当前) │ D1_new = feedback⊕D2⊕D3 │
  ├──────┼─────────────────────────┼───────────┼───────────┼─────────────────────────┤
  │ 1    │ s2⊕s3                   │ s2        │ s3        │ (s2⊕s3)⊕s2⊕s3 = 0       │
  ├──────┼─────────────────────────┼───────────┼───────────┼─────────────────────────┤
  │ 2    │ s1⊕s2                   │ 0        │ s2        │ (s1⊕s2)⊕s1⊕s2 = 0       │
  ├──────┼─────────────────────────┼───────────┼───────────┼─────────────────────────┤
  │ 3    │ 0⊕s1                    │ 0         │ 0        │ s1⊕0⊕s1 = 0             │
```

---

#### Q3: 交织器是怎么实现的？

Turbo 编码的精髓在于交织器——把数据打乱顺序再编码一遍。LTE 用的是 QPP（二次置换多项式）交织器。

**数学定义（TS36.212 5.1.3.2.3）：**

```
Π(i) = (f1 × i + f2 × i²) mod K

其中: i = 0, 1, 2, ..., K-1 (原始顺序)
      Π(i) = 写地址 (打乱后的位置)
      f1, f2 = 取决于K的常量（查表5.1.3-3）
```

比如 K=6144，f1=263, f2=480：
```
i=0:   Π(0) = (263×0 + 480×0) mod 6144 = 0
i=1:   Π(1) = (263×1 + 480×1) mod 6144 = 743
i=2:   Π(2) = (263×2 + 480×4) mod 6144 = 2446
```

原始第 0 个bit写到地址 0，第 1 个bit写到地址 743，第 2 个bit写到地址 2446……数据被彻底打乱。

**硬件怎么算？——迭代法避免乘法器**

如果每周期算 i² × f2，需要一个硬件乘法器（在 245.76MHz 下延迟大且面积大）。硬件用的是**迭代递推+纯加法**：

定义辅助序列 f, g, h, i，每周期递推一次：
- f(i) 是写地址
- g(i) = f(i+1) - f(i)（一阶差分）
- h(i) = g(i+1) - g(i)（二阶差分）
- i(i) 是索引增量

每次递推只需要**加法**，不需要乘法。核心子模块 `QPP_Interleaver_Calculation_Core` 被例化了 4 个实例，分别计算 (f,g)、(g,h)、(h,i)、(i,delta_ib) 的递推：

```verilog
QPP_Interleaver_Calculation_Core iteration_calculation_fg(
    .fb_add_gb(fb_add_gb),          // 上一周期的 fb+gb
    .S(K_div_P),                     // K/4 = 1536 (归一化因子)
    .f_add_g_minus_S(f_add_g_minus_S), // f+g 减去 S
    .f_add_g(f_add_g),              // f+g
    .fb(fb_feedback),               // 更新后的 fb (用于下一周期)
    .f(f_feedback)                  // 更新后的 f (= 写地址)
);
```

**初始化参数从哪来？**

`Get_Code_Block_Size_Table_Index` 把 K（如 6144）映射为 index（0~187），然后 case 语句查出 33-bit 的递推初始参数：

```
calculation_parameter:
  [32:31] = delta_ib
  [30:29] = ib
  [28:27] = hb
  [26:25] = gb
  [24:22] = i_index (对应 i 的初始值: 0/624/432/656/688/752)
  [21:11] = h (初始值)
  [10:0]  = g (初始值)
```

reset 时加载这些初始值，之后每个 enable 周期纯加法递推。最终 `write_address = f[10:0]`。

**地址分配：4 个存储体**

写地址的低 2 位（通过 fb 计算）被用作 `memory_select`（0~3），分配到 4 个存储体：
- Bank 0: offset = 0
- Bank 1: offset = K/4
- Bank 2: offset = 2K/4
- Bank 3: offset = 3K/4

4 个体可以并行操作，提高吞吐。

**写入端 vs 读取端：**

```
写入端 (Interleaver_Write_Stream_Generator):
  用 QPP 公式计算 Π(i) → 按打乱地址写入 Interleaver_Buffer
  数据bit按原始顺序进来，存到打乱的位置

读取端 (Interleaver_Read_Stream_Generator):
  按 0, 1, 2, ..., K-1 顺序读取
  读出的是按照 QPP 打乱后的数据
```

**完整时序：**

```
周期 0:  new_code_block=1 → reset → 加载初始参数
周期 1:  bit_in_valid=1 → 第1次递推，算出 Π(0)
周期 2:  写地址延迟1拍 → Interleaver_Buffer 写端口锁存
...
周期 K:  iteration_start=1 → 读地址生成器复位
周期 K+1: read_enable=1 → 从地址0开始顺序读出
```

---

#### Q4: 线性编码和交织编码用同一个还是两个 Turbo_Encoder？

在 `Turbo_Encoder_Top` 中，例化了**两个独立的 `Turbo_Encoder` 硬件实例**：

```verilog
// 第1个: 编码原始顺序的数据 → d0 + d1
Turbo_Encoder encode_input_bit(
    .bit_in(bit_in_delay7),      // 原始bit (延迟7周期等交织器)
    .x_out(x_out),                // → d0
    .z_out(z_out),                // → d1
);

// 第2个: 编码交织后的数据 → d2
Turbo_Encoder encode_interleaver_bit(
    .bit_in(interleaver_bit),    // 从 Interleaver_Buffer 读出的bit
    .x_out(interleaver_x_out),   // → 交织系统位 (不输出)
    .z_out(interleaver_z_out),   // → d2
);
```

它们**不是分时复用的，是两套独立硬件**，各有各的 `Turbo_Encoder_Filter` 和 `num_encoded_bits` 计数器。

原因是两轮编码在时间上有重叠——第一轮跑完 K+3 个周期出 d0+d1 的同时，第二轮已经开始从交织器读数据编码了。两个独立实例可以完全流水线化。Turbo_Encoder 很小（3 个寄存器 + 少量组合逻辑），多花一份面积换吞吐量，完全值得。

#### 6a-4. TX_Rate_Matcher — 速率匹配：删减/重复bit使之恰好填满分配的RE

**解决的问题：** Turbo编码输出了 3×(K+4) 个bit，但无线资源只分配了 E 个bit。速率匹配器负责把 3Kπ 个bit变成恰好 E 个bit。

**三步标准化流程（TS36.212 5.1.4）：**

```
d0(K+4 bit) → [子块交织(32列矩阵)] → d0'  ┐
d1(K+4 bit) → [子块交织(32列矩阵)] → d1'  ├→ [bit收集] → 循环缓冲区(Kw=3Kπ)
d2(K+4 bit) → [子块交织(32列矩阵)] → d2'  ┘       ↓
                                            从k0位置开始，顺序读E个bit → bit_out
```

**硬件实现（4个子模块协作）：**

```
                    ┌─ [Rate_Matcher_Interleaved_Address_Generator #1] ─┐
  data_0 ──────────→│  mode=01 (S/P1), 生成子块交织写地址(S和P1)        │
  data_0_valid ────→│                                                    │
                    └────────────────────────────────────────────────────┤
                    ┌─ [Rate_Matcher_Interleaved_Address_Generator #2] ─┐│
  data_1_2 ────────→│  mode=10 (P2), 生成子块交织写地址(P2)             ││
  data_1_2_valid ──→│                                                    ││
                    └────────────────────────────────────────────────────┤│
                                                                         ↓↓
                    ┌──────────────────────────────────────────────────────┐
                    │  TX_Rate_Matcher_Circular_Buffer (双页Ping-Pong)     │
                    │  写端口: d0→S区, d1→P1区, d2→P2区                   │
                    │  读端口: 从k0开始顺序读E个bit (跨S/P1/P2区读取)      │
                    └──────────────────────────────────────────────────────┘
                                              ↑ 读地址来自
                    ┌──────────────────────────────────────┐
                    │ Rate_Matcher_Linear_Address_Generator │
                    │ 从k0开始，到(k0+E-1) mod Kw          │
                    └──────────────────────────────────────┘

  总控: TX_Rate_Matcher_FSM — 管理page切换、读写时序
```

**双页（Ping-Pong）设计：**

这是为了吞吐量优化的关键设计。循环缓冲区有两个 Page（Page 0 和 Page 1），写和读在不同的 Page 上并行进行：
- 当 Turbo 编码器在往 Page 0 写码块 N 的数据时
- 速率匹配器同时在从 Page 1 读码块 N-1 的数据

`page_select` 信号由 FSM 控制，每处理完一个码块就翻转。这避免了"写完再读"的串行等待。

---

### 分支2：PXSCH_Deserializer_Scrambler_Modulator — 加扰调制链

#### 子模块连接关系

```
  encoding_bit_out ──→ [delay 33] ──→ [PXSCH_Deserializer_Bit_to_Symbol] ──→ [PXSCH_Scrambler] ──→ [LTE_Modulation] ──→ I/Q
  valid_in ──────────→ [delay 33] ──→                                     ──→                    ──→
                                       串并转换(1周期)                      Gold加扰(1周期)       星座映射(2周期)
                                       
  new_control_pulse ──→ [delay 1] ──→ [Scrambler_Cinit_to_X1_X2] (33周期)
                                      │ c_init = cell_ID | RNTI<<14 | subframe<<9
                                      │ ROM查表 + 逐bit迭代 → x1_init, x2_init
                                      ↓ (33周期后)
                                      x1_init[31:0], x2_init[31:0]
                                                ↓
                                      [PXSCH_Scrambler] 加载为初始状态
```

**总延迟 = 33(c_init) + 1(串并转换) + 1(加扰) + 2(调制) + 1(顶层输出打拍) = 37周期**（不含编码链）

**为什么是33周期？** 因为 c_init 有31个bit要逐bit和ROM值做异或，每个bit一个周期 = 31周期，加2周期流水线 = 33。

**为什么输入数据要等33周期？**
```verilog
delay_1 #(33) delay_PXSCH_Scr_Mou_bit_in(.In(bit_in), .Out(bit_in_delay33));
delay_1 #(33) delay_PXSCH_Scr_Mou_valid_in(.In(valid_in), .Out(valid_in_delay33));
```
因为编码链输出的bit和c_init计算是同时开始的——trigger一来，编码链开始跑，c_init也开始算。c_init算好之前不能加扰（x1/x2初始值还没准备好），所以bit先排队等33拍。

---

#### 6b-1. Scrambler_Cinit_to_X1_X2 — 计算Gold序列的初始种子

**解决的问题：** LTE 加扰用的 Gold 序列 = x1序列 ⊕ x2序列。每来一个新的子帧，x1和x2都要重新初始化。x1的初始值是固定的（0x5E485840），但x2的初始值取决于 c_init。

**c_init 的计算（在顶层 Deserializer_Scrambler_Modulator 中）：**
```verilog
c_init_value <= {16'd0, cell_ID}           // bit[15:0] = cell_ID
              | ({16'd0, RNTI} << 4'd14)   // bit[29:14] = RNTI
              | ({24'd0, subframe_index} << 4'd9);  // bit[12:9] = subframe_index
```

**x2初始值的逐bit迭代计算（核心算法）：**
```verilog
// 初始 x2_init = 0
// 对 i = 0 到 30:
//   x2_init_new = x2_init_previous | ( xor(c_init & ROM[i]) << i )

x2_init_value_temp <= (x2_init_previous) | (x2_init_xor_c_init << index_out_delay1);
```

`ROM_for_scrambler_X2` 里存了31个32-bit值，每个值和 c_init 按位与，然后逐位异或得到一个bit，这个bit放在 x2_init 的第 i 位。

PULSE_TO_STROBE_U16 生成 index 从 0 到 30，每个index读一次ROM，做一个异或，移位，或到结果里。31个周期完成。

**x1初始值：** 固定 `32'd1581799488`（= 0x5E485840），这是LTE协议规定的。

---

#### 6b-2. PXSCH_Deserializer_Bit_to_Symbol — 串行bit拼成并行符号

**解决的问题：** Turbo编码输出是1bit/cycle的串行流，但调制器需要2/4/6bit一组的符号。

**核心算法——逐bit修改法：**

不是用移位寄存器等待凑够，而是维护一个"当前符号"，每次修改其中一位：

```verilog
// symbol_out_next = 当前正在构建的符号
// bit_index = 当前要修改第几位 (0→2, 0→4, 0→6)

// 检查：要修改的bit和当前符号中的对应bit是否相同？
change_flag = (symbol_out_next[bit_index] != bit_in);

// 如果不同，翻转该位
symbol_out_current = change_flag ? (symbol_out_next ^ (6'd1 << bit_index)) : symbol_out_next;

// 凑够了或最后一bit？
stop = (bit_index_add1 == bits_per_symbol) | last_sample_in;

// 凑够了→输出，归零。没凑够→继续。
symbol_out_next <= stop ? 6'd0 : symbol_out_current;
valid_out_reg <= stop & valid_in;
```

举个例子（QPSK，bits_per_symbol=2）：
```
bit_in: 1, 0, 1, 1, 0, 0, ...
        ↓  ↓  ↓  ↓  ↓  ↓
cycle1: idx=0, sym=0→bit0翻1→sym=01, stop=0 (idx+1=1≠2)
cycle2: idx=1, sym=01→bit1翻0→sym=01, stop=1 (idx+1=2=2) → 输出01, sym归0
cycle3: idx=0, sym=0→bit0翻1→sym=01, stop=0
cycle4: idx=1, sym=01→bit1翻1→sym=11, stop=1 → 输出11, sym归0
cycle5: ...
```

这个设计的优雅之处：不需要移位寄存器，只需要1个6-bit寄存器 + 1个3-bit索引，每个周期只修改1个bit。

---

#### 6b-3. PXSCH_Scrambler — Gold序列加扰

**解决的问题：** 对每个调制符号的bit做随机化（加扰），避免长串的0或1导致频谱不平坦。

**本质** = 调制符号 ⊕ Gold序列的低 N 位。

```verilog
// 组合逻辑 Scrambler（无延迟）:
xor_result = data_in ^ x1_state[7:0] ^ x2_state[7:0];  // 8-bit异或
// 根据调制方式截断高位
data_out = xor_result & bit_mask;  // QPSK保留低2位，16QAM保留低4位，64QAM保留低6位
```

**x1/x2状态更新（关键：每次消耗N步）：**

x1和x2都是31阶LFSR（线性反馈移位寄存器）。正常每周期递进1步。但这里每个符号消耗 N 个bit的Gold序列（QPSK=2, 16QAM=4, 64QAM=6），所以 x1/x2 要一次性递进 N 步。

`Scrambler_Process_X1` 和 `Scrambler_Process_X2` 是组合逻辑，输入当前状态 + 递进步数，输出递进后的状态。没有时钟延迟。

**状态更新逻辑：**
```verilog
if(scrambler_initialization_valid)
    x1_state_reg <= x1_initial_value;  // 加载新初始值（新子帧开始）
else if(valid_in)
    x1_state_reg <= x1_state_out;      // 递进N步
else
    x1_state_reg <= x1_state_reg;      // 保持
```

---

#### 6b-4. LTE_Modulation — QPSK/16QAM/64QAM星座映射

**解决的问题：** 把6-bit的加扰后符号映射到 I/Q 复数平面的某个点。

**内部结构：**
```
constellation_bits[7:0]
        │
   ┌───→ [Map_LTE_to_Common_Modulation_Symbol] (LTE比特重排, 1周期)
   │         LTE和802.11的星座bit顺序不同，此处做重映射
   │
   ├───→ [QPSK_Modulation] (2周期)
   │         输入2bit → 查表 → I/Q fix16_15
   │
   ├───→ [QAM16_Modulation] (2周期)
   │         输入4bit → 查表 → I/Q fix16_15
   │
   └───→ 64QAM (通过 modulation 选择)
             输入6bit → 查表 → I/Q fix16_15
```

根据 `modulation` 的值，只有一路的 input_valid 为1，其他为0。输出用 OR 合并：
```verilog
output_valid <= QPSK_output_valid | QAM16_output_valid;
symbol_out_real = QPSK_real | QAM16_real;  // 未激活的为0，所以OR=选择
symbol_out_imag = QPSK_imag | QAM16_imag;
```

**fix16_15 定点格式：**
- 16-bit：1位符号 + 1位整数 + 14位小数
- 范围：-2.0 ~ +1.9999
- QPSK：±0.7071 ≈ ±23170/32768
- 16QAM：±0.3162, ±0.9487（两种幅值）
- 64QAM：±0.3086, ±0.6171, ±0.9258（三种幅值）

---

### 完整时序：一个TB=30576从trigger到I/Q输出

以最大TB=30576(C=5, K=6144, E=12480)为例，这是整个模块最忙的场景：

```
T=0:        PXSCH_transmitter_trigger ↑
            ├→ modulation, TB_size, subframe_index 锁存到内部寄存器
            ├→ PXSCH_transmitter_trigger_delay1 → Channel_Encoder 启动
            └→ Scrambler_Cinit_to_X1_X2 开始计算 (33周期)

T=1~33:     c_init计算中 (ROM逐bit迭代)
            编码链: Parameter_Computation 查表中 (98周期后出结果)

T=33:       c_init计算完成 → x1_init, x2_init 就绪
            new_control_pulse_out=1 → Scrambler 加载初始值

T=34~98:    Scrambler 就绪，等待编码链输出
            编码链: Parameter_Computation 还在等98周期延迟

T=98:       valid_out_para=1 → CRC_Add_and_CB_Segmentation 收到参数
            → 状态: IDLE → Wait_for_Ready_Flag
            → 等待 Turbo_Encoder_Top 的 ready_for_output

T=~99:      ready_for_output=1 → CRC状态机进入 Write_Input_Data
            → ready_for_input=1 → MAC_FIFO 开始发bit

T=99~30674: CRC 逐bit接收 30576 bit + 实时算 TB_CRC
            (每隔 6144 bit，C>1时进入Write_CB_CRC)
            Turbo编码+速率匹配也在并行工作（流水线重叠！）

T=~30774:   全部30576bit + CRC处理完毕
            → CRC状态机回到 IDLE

T=~30774~   PXSCH_Deserializer_Scrambler_Modulator 处理编码后的bit
            (每个编码bit延迟37周期后变成I/Q符号)

T≈30774+62400×5+37 = ~342811:
            最后一个符号从 symbol_valid 输出
```

**实际 wall-clock 延迟远小于上述之和**，因为编码链内部是深度流水线——CRC还在收TB数据时，已经收完的码块已经在做 Turbo 编码和速率匹配了。ENCODE_SAFE_VALUE=20000 是稳态延迟。

---

### 反压链路总结

```
Rate_Matcher "缓冲区满" → ready_for_code_block=0
  → (delay 3) → Turbo_Encoder_Top.ready_for_code_block=0
  → CRC_Add_and_CB_Segmentation.ready_for_output=0
  → CRC_Add_and_CB_Segmentation.ready_for_input=0
  → Channel_Encoder.ready_for_data=0
  → 顶层 ready_for_input=0
  → MAC_FIFO 状态机暂停 → read_FIFO_flag=0
  → Buffer_UDP_Data 停止读出 → FIFO可能满 → UDP停写
```

每一级都是反压的传播节点，保证数据不会丢失，也不会溢出。

---

## 校正：PXSCH_TX_Bit_Processing_Top 重新精读（2026-08-09）

> 本节以当前 RTL 的实际连接为最高证据，校正前文中对总输出长度、固定延时、64QAM 和反压链的部分错误结论。旧内容保留，用于观察理解过程；发生冲突时以本节为准。

### 1. 一句话定位

`PXSCH_TX_Bit_Processing_Top` 不是单独的 Turbo 编码器，而是 PDSCH 发送侧“比特流到调制符号”的编排顶层：

```text
MAC_TX 串行 payload bit
    → 参数查表
    → TB/CB CRC 与码块分段
    → Turbo 编码
    → 速率匹配
    → Qm 个 bit 组成一个调制字
    → PDSCH 加扰
    → QPSK/16QAM 映射
    → Q2.14 复数调制符号

同时：
原始 OFDM-symbol trigger
    → 每个触发独立延迟 ENCODE_SAFE_VALUE 个周期
    → 送给后续资源映射模块
```

源码位置：`PXSCH_TX_Bit_Processing_Top.v:25-211`；在 `TX_BIT_Processor.v:142-157` 中例化。虽然模块名使用通用的 `PXSCH`，但本工程发送链给它的是 `PDSCH_transmitter_trigger`，所以当前用途是下行 PDSCH。

### 2. 顶层端口与真正含义

| 端口 | 位宽/方向 | 单位 | 当前作用 |
|---|---:|---|---|
| `clk_192M` | 1/input | cycle | 实际连接到 `TX_BIT_Processor.clk`，即 `clk_baseband`；频率不能只根据端口名断言 |
| `rst_n` | 1/input | — | 低有效复位；并非所有内部延迟寄存器都接复位 |
| `MCS_data` | 1/input | payload bit | 名字容易误导；它不是 MCS 参数，而是 MAC 输出的串行 TB 比特 |
| `MCS_valid` | 1/input | bit-valid | `MCS_data` 当前拍是否有效 |
| `PXSCH_transmitter_trigger` | 1/input | pulse | 启动一次配置，同时锁存本次 TB 的参数 |
| `OFDM_symbol_trigger_in` | 1/input | pulse | 原始 OFDM 符号边界触发 |
| `start_of_radio_frame` | 1/input | pulse | 当前只进入一段遗留延时/下降沿检测逻辑，没有影响任何顶层输出 |
| `modulation` | 2/input | enum | `1=QPSK`、`2=16QAM`、`3=64QAM`；但当前真正实现只到 16QAM |
| `subframe_index` | 4/input | subframe | 锁存后参与扰码 `c_init`；同时选择 `RE_NUMBER0/1` |
| `transblock_size` | 17/input | bit/TB | 本次传输块的 payload 比特数 |
| `delay` | 32/input | cycle | OFDM 触发的计划延迟；当前上层接 `ENCODE_SAFE_VALUE=20000` |
| `OFDM_symbol_trigger_out` | 1/output | pulse | 延迟后的 OFDM 符号边界 |
| `symbol_valid` | 1/output | symbol-valid | `symbol_real/imag` 当前拍有效 |
| `ready_for_input` | 1/output | ready | 编码链是否允许 MAC 继续送 payload bit |
| `symbol_real/imag` | 16+16/output | Q2.14 | 调制符号的实部和虚部 |

### 3. 完整调用树

```text
PXSCH_TX_Bit_Processing_Top
├── delay_1：PXSCH trigger 延迟 1 拍
├── 参数锁存：modulation / subframe / RE number / TB size
├── PXSCH_Channel_Encoder
│   ├── PXSCH_Parameter_Computation：按 TB size 查编码与速率匹配参数
│   ├── CRC_Add_and_CB_Segmentation：TB CRC、码块分段、CB CRC
│   ├── Turbo_Encoder_Top：1/3 母码率 Turbo 编码
│   └── TX_Rate_Matcher：交织、循环缓冲区读取和速率匹配
├── PXSCH_Deserializer_Scrambler_Modulator
│   ├── delay_1#(33)：等待扰码初值生成
│   ├── Scrambler_Cinit_to_X1_X2：生成 x1/x2 初始状态
│   ├── PXSCH_Deserializer_Bit_to_Symbol：每 Qm bit 组成一个调制字
│   ├── PXSCH_Scrambler：与扰码序列异或
│   └── LTE_Modulation
│       └── Modulation：当前只有 QPSK 和 16QAM 两套有效实例
├── Trigger_Delay_10x：实际包含 14 个并行触发延迟槽
├── delay_n#(1,32)：最终 I/Q 再延迟 1 拍
├── delay_1#(1)：最终 symbol_valid 再延迟 1 拍
└── start_of_radio_frame 遗留支路：当前结果未被使用
```

### 4. 为什么 trigger 要延迟 1 拍

在原始 `PXSCH_transmitter_trigger` 到达的 T0 上升沿，顶层先锁存：

```verilog
modulation_delay1   <= modulation;
subframe_index_delay1 <= {4'd0, subframe_index};
number_of_REs_delay1  <= (subframe_index == 0) ? RE_NUMBER0 : RE_NUMBER1;
transblock_size_delay1 <= transblock_size;
```

同一个 trigger 又经过 `delay_1#(1)`，到 T1 才送给：

- `PXSCH_Channel_Encoder.configuration_valid_in`；
- `PXSCH_Deserializer_Scrambler_Modulator.new_control_pulse`。

因此 T1 启动子模块时，T0 锁存的参数已经稳定。这一拍的作用是“配置与启动对齐”，不是把复杂组合逻辑切成一级流水。

### 5. 数据路径与控制路径

数据路径：

```text
MCS_data/MCS_valid
 → PXSCH_Channel_Encoder
 → encoding_bit_out/valid
 → 延迟33拍并按 Qm 聚合
 → 加扰
 → QPSK/16QAM
 → modulation_real/imag/valid
 → 顶层统一延迟1拍
 → symbol_real/imag/valid
```

控制路径：

```text
PXSCH_transmitter_trigger(T0)
 ├→ 锁存 modulation、subframe、TB size、RE number
 └→ delay 1
      ├→ 启动编码参数查表
      └→ 启动扰码初值计算

OFDM_symbol_trigger_in
 → Trigger_Delay_10x(delay=20000)
 → OFDM_symbol_trigger_out
```

`ready_for_input` 直接来自 `PXSCH_Channel_Encoder.ready_for_data`，再反馈给 `MAC_TX.ready_for_output`。它控制的是“MAC 是否继续给编码器送 bit”，并不等同于 UDP 接收端已经获得完整的端到端反压。

### 6. 编码输出长度不是动态按 RE×Qm 计算

顶层确实计算并传入：

```verilog
number_of_REs_delay1 <= subframe0 ? 10800 : 15600;
```

但当前 `PXSCH_Channel_Encoder.v` 中：

- `number_of_REs` 只出现在端口声明中；
- `modulation` 只出现在端口声明中；
- `redundancy_version_index` 也只出现在端口声明中。

三者都没有进入参数计算或速率匹配实例。真正的 `E+`、`E-`、码块数 C、K、k0 等，是 `PXSCH_Parameter_Computation` 仅根据 `transport_block_size` 进行硬编码查表得到的。

因此总速率匹配输出长度应计算为：

```text
G = 各码块 E 的总和
若所有码块 E 相同：G = C × E
调制符号数 = G ÷ Qm
QPSK: Qm=2；16QAM: Qm=4
```

结合当前 `MCS_to_TBS` 和参数表：

| 子帧 | MCS_control | 调制 | TBS | C×E = G(bit) | G/Qm(symbol) | 配置的RE | 结论 |
|---|---:|---|---:|---:|---:|---:|---|
| 0 | 0 | QPSK | 7224 | 2×10800=21600 | 10800 | 10800 | 闭合 |
| 0 | 1 | QPSK | 8760 | 2×10800=21600 | 10800 | 10800 | 闭合 |
| 0 | 2 | QPSK | 15840 | 3×7200=21600 | 10800 | 10800 | 闭合 |
| 0 | 3 | QPSK | 8760 | 2×10800=21600 | 10800 | 10800 | 闭合 |
| 非0 | 0 | QPSK | 10296 | 2×15600=31200 | 15600 | 15600 | 闭合 |
| 非0 | 1 | QPSK | 15840 | 3×7200=21600 | 10800 | 15600 | **少4800个RE，需验证** |
| 非0 | 2 | QPSK | 29296 | 5×6240=31200 | 15600 | 15600 | 闭合 |
| 非0 | 3 | 16QAM | 30576 | 5×12480=62400 | 15600 | 15600 | 闭合 |

这里暴露出一个结构问题：`TBS=15840` 同时用于“子帧0/MCS_control=2”和“非0子帧/MCS_control=1”，但参数查表只看 TBS，无法根据 RE 数区分两个场景。源码中曾注释掉 `E=10400` 的版本，即 `3×10400=31200 bit`，这恰好对应非0子帧的 15600 个 QPSK RE；当前生效值却是 `E=7200`，只对应10800个 QPSK RE。因此这一 MCS 组合不能宣称已正确填满非0子帧。

### 7. 对前文 TB=30576 时序的校正

前文写成了：

```text
62400 × 5
```

这是把“5个码块的总输出”又乘了一次码块数。正确关系是：

```text
C = 5
每码块 E = 12480 bit
G = C × E = 5 × 12480 = 62400 coded bit
16QAM符号数 = 62400 ÷ 4 = 15600 symbol
```

所以不能再使用“最后输出约为 `30774 + 62400×5 + 37`”这一旧估算。编码、CRC、Turbo 和速率匹配之间还有流水重叠，若要得到 trigger 到首/末符号的精确周期数，必须用 testbench 记录实际握手和 valid，不能只把各段长度相加。

### 8. “37周期固定延时”的校正

`PXSCH_Deserializer_Scrambler_Modulator.v` 的旧注释写“延时为37”，但当前 RTL 结构是：

- 输入 bit/valid 固定延迟 33 拍；
- bit-to-symbol 要等待 Qm 个有效 bit 聚齐，因此首符号等待与 Qm 有关；
- 加扰 valid 延迟 1 拍；
- 当前 `LTE_Modulation` 的代码和注释均显示 4 拍；
- 顶层最终再把 I/Q 与 valid 同步延迟 1 拍。

因此不能把整个模块简单概括成“每个编码 bit 37 拍后变成 I/Q”。输入和输出的处理粒度不同：输入是一拍一个 coded bit，输出是一拍一个 modulation symbol；QPSK 每2 bit出1符号，16QAM每4 bit出1符号。精确首符号延迟应通过波形确认。

### 9. 当前没有真正实现64QAM

虽然多个端口和 `case` 都定义了 `QAM64=3`，而且 bit 聚合器能够收集6 bit，但 `Modulation.v` 当前只例化：

- `QPSK_Modulation`；
- `QAM16_Modulation`。

`output_valid` 也只等于两者 valid 的 OR。若输入 `modulation=3`，两路 input-valid 都不会拉高，最终不会产生有效的64QAM符号。因此“枚举支持64QAM”不等于“数据通路已实现64QAM”。另外，当前上游 `CM_to_PDSCH_Encoder` 只会选择 QPSK 或16QAM，本身也不会生成3。

### 10. ENCODE_SAFE_VALUE=20000 的准确含义

`ENCODE_SAFE_VALUE` 并不是整个信道编码的总 latency。它只接到 `Trigger_Delay_10x.delay`，用于把每一个 `OFDM_symbol_trigger_in` 延后约20000个基带时钟周期，让编码和调制数据先跑一段时间，再启动后续资源映射的 OFDM 时间线。

这里没有 `ready/valid` 握手来动态决定何时释放 OFDM trigger，所以它属于固定安全窗口。20000是否对全部 TBS/MCS 都足够，需要比较波形中的：

```text
PXSCH_transmitter_trigger
encoding_bit_out_valid 首拍
symbol_valid 首拍
OFDM_symbol_trigger_out 首拍
```

若实际基带时钟确认为192 MHz，20000周期约为104.17 μs；在时钟源未核实时，只应写“20000个基带时钟周期”。

### 11. Trigger_Delay_10x 的真实结构和风险

模块名字及旧注释说“10个触发”，当前 RTL 实际具有：

- 14位 `array/strobe/falling`；
- 模14的触发槽索引；
- 14个 `PULSE_TO_STROBE_U32` 实例。

其用途是允许多个 OFDM 符号触发在延时窗口内同时挂起。实际代码使用 `delay_minus_2 <= delay - 3`，变量名、注释和运算值并不一致；`delay<3` 还会发生32位无符号下溢。此外，下降沿检测 `else` 后缺少 `begin/end`，只有 `falling[0]` 真正受复位分支控制，其余位每拍都会执行。再叠加公共 `delay_1/delay_n` 使用阻塞赋值，精确是否恰好延迟 D 拍必须通过仿真确认，暂不能只凭注释定论。

### 12. 顶层中的遗留/无效逻辑

当前源码确认：

- `start_of_radio_frame` 被延迟并展宽，再检测下降沿；但检测结果 `start_of_radio_frame_delay1_strobe_falling` 没有被任何输出或控制使用；
- `number_of_subframe_latch` 和 `number_of_subframe_out_reg` 只有声明，没有读写；
- `number_of_REs_delay1` 虽然锁存并传入编码器，但编码器内部没有使用；
- `last_sample_in` 被顶层固定为0，编码结束依赖 valid 数量/资源计划，而不是显式 last；
- `subframe_index_delay1`、`number_of_REs_delay1`、`transblock_size_delay1` 没有复位分支，第一次 trigger 前仿真会是未知值；配置启动前被锁存后通常才进入有效数据路径，但仍应注意仿真可见的 X。

### 13. 当前工程集成处还有“注释吞连接”问题

`TX_BIT_Processor.v` 当前文本中多处本应换行的内容粘到了 `//` 注释之后，导致以下端口在 Verilog 语义上被注释掉：

- `MAC_TX.MAC_trigger`；
- `PXSCH_TX_Bit_Processing_Top.start_of_radio_frame`；
- `PXSCH_TX_Bit_Processing_Top.OFDM_symbol_trigger_out`；
- `PXSCH_TX_Bit_Processing_Top.symbol_valid`；
- `PXSCH_TX_Bit_Processing_Top.symbol_real`。

只有 `ready_for_input` 和 `symbol_imag` 等仍在有效代码行中。这意味着“模块本体的设计功能”和“当前 `TX_BIT_Processor` 实际集成效果”必须分开看；按当前文本直接综合/仿真，PDSCH链不会按预期工作。`PXSCH_Parameter_Computation.v` 也存在 `assign ready_for_configuration`、`sync_latch.out` 被同行注释吞掉的现象。下一步若要做系统波形，必须先确认这些文件是不是损坏副本，并恢复端口换行。

### 14. 反压结论的校正

能够由当前连接确认的反压链是：

```text
TX_Rate_Matcher
 → Turbo_Encoder_Top
 → CRC_Add_and_CB_Segmentation.ready_for_input
 → PXSCH_Channel_Encoder.ready_for_data
 → PXSCH_TX_Bit_Processing_Top.ready_for_input
 → MAC_TX.ready_for_output
```

它最多证明 MAC 向编码器的 bit 输出会暂停。不能直接推出“UDP一定停写、FIFO一定不满、端到端绝不丢数据”。只有当 FIFO `full/almost_full` 真正连回 UDP 源的写使能或 ready 时，才能形成完整写侧反压。原文“每一级保证不会溢出”和“UDP停写”结论过强，应撤回。

### 15. 建议的最小仿真观察点

为了把本节的待验证项变成源码确认，建议只做一次最小事务并记录：

```text
PXSCH_transmitter_trigger
modulation_delay1 / transblock_size_delay1 / subframe_index_delay1
PXSCH_transmitter_trigger_delay1
ready_for_input / MCS_valid / MCS_data
encoding_bit_out_valid / encoding_bit_out
bit_to_symbol_valid
scrambler_valid
modulation_valid
symbol_valid / symbol_real / symbol_imag
OFDM_symbol_trigger_in / OFDM_symbol_trigger_out
```

同时设置三个计数器：

1. `encoding_bit_out_valid` 的总拍数，验证是否等于 G；
2. `symbol_valid` 的总拍数，验证是否等于 G/Qm；
3. 每个 OFDM trigger 输入到输出的周期差，验证 `Trigger_Delay_10x` 的真实延迟。

---

## 待读模块

- [x] 第4级: Buffer_UDP_Data (异步FIFO跨时钟域) — 2026-08-06
- [x] 第5级: MAC_TX / MAC_FIFO (六状态机组帧) — 2026-08-07
- [x] 第6级: PXSCH_TX_Bit_Processing_Top (Turbo编码+加扰+调制) — 2026-08-07
- [ ] 第7级: PDCCH_Transmitter_Top (控制信道)
- [ ] 第8级: Combine_Control_and_Data (资源映射+WFRFT)
- [ ] 第9级: Invalidate_Layer_Streams (非活跃层置零)

---

---
*最后更新: 2026-08-09*
