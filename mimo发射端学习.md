> 范围：以 `MIMO_TX_Top` 为发射端总入口，只记录已经从当前 RTL 核实的结论。
>
> 标题规则：一级标题对应 `MIMO_TX_Top` 的直接子模块；二级标题对应各模块内部的直接子模块；每个模块统一按“实现功能、实现原理解读、子模块关系图、关键代码解读、讨论问题”组织。
>
> 证据优先级：当前有效 RTL 连接 > 参数宏和 IP 配置 > 源码注释 > 历史讨论。源码注释与连接冲突时，以有效代码为准。

发射端总链路：

```text
3路UDP用户数据
 → 3×TX_BIT_Processor
 → TX_BIT_FIFO_Exchange
 → FIFO_Manager/Aurora跨板交换
 → TX_MIMO_Processor
 → TX_RRH_FIFO_Exchange
 → TX_RRH_Processor
 → Over_Sample_Group
 → DAC/RF
```

当前 `MIMO_Parameter.vh` 的关键规模：

```text
FPGA_ID                  = 3
NUMBER_LAYER             = 12
NUMBER_LAYER_PER_FPGA    = 3
NUMBER_ANTENNA           = 32
SUBCARRIER_PER_RB        = 12
SUBCARRIER_PER_OFDM      = 3600
OFDM_SYMBOL_PER_SLOT     = 7
OFDM_SYMBOL_PER_RADIO10  = 140
```

# 1. TX_BIT_Processor

## 1.0 TX_BIT_Processor 顶层

### 实现功能

`TX_BIT_Processor` 完成单个本地 layer 的发送侧 bit 处理：接收一路 UDP 字节流和系统控制参数，最终输出一条按 OFDM 网格组织的复数基带流。

`MIMO_TX_Top` 一共例化三个该模块：

```text
Layer_ID = FPGA_ID×3 + 0
Layer_ID = FPGA_ID×3 + 1
Layer_ID = FPGA_ID×3 + 2
```

当前 `FPGA_ID=3`，因此本 FPGA 对应 layer 9、10、11；四块 FPGA 合计12层。

### 实现原理解读

它同时维护三类路径：

```text
时间路径：无线帧触发 → 符号/子帧/子载波计数
数据路径：UDP字节 → 自定义MAC成帧 → 信道编码 → 加扰调制
网格路径：PDCCH/PDSCH/RS → 资源映射 → 变换输出
```

三条路径不能只比较模块级延时，必须通过 `valid`、trigger 和固定安全窗口在后续重新对齐。

### 子模块关系图

```text
TX_BIT_Processor
├─ TX_Trigger
├─ TTI_Handing_Top
├─ TX_PDSCH_Configuration
├─ Buffer_UDP_Data
├─ MAC_TX
├─ PXSCH_TX_Bit_Processing_Top
├─ PDCCH_Transmitter_Top
├─ Combine_Control_and_Data
└─ Invalidate_Layer_Streams
```

### 关键代码解读

直接实例位置：

| 子模块 | `TX_BIT_Processor.v` 行号 | 主要输出 |
|---|---:|---|
| `TX_Trigger` | 36 | 原始 OFDM symbol trigger |
| `TTI_Handing_Top` | 59 | 四级索引、CM message |
| `TX_PDSCH_Configuration` | 84 | PDSCH trigger、MCS/TBS |
| `Buffer_UDP_Data` | 112 | 基带时钟域字节流 |
| `MAC_TX` | 127 | 串行 TB bit |
| `PXSCH_TX_Bit_Processing_Top` | 150 | PDSCH I/Q 符号 |
| `PDCCH_Transmitter_Top` | 176 | PDCCH I/Q 符号 |
| `Combine_Control_and_Data` | 192 | 资源映射后的复数流 |
| `Invalidate_Layer_Streams` | 222 | 使能门控后的 layer 流 |

### 讨论问题

1. 当前 `TX_BIT_Processor.v`、`MAC_FIFO.v` 等文件存在中文乱码后换行丢失，部分端口连接或状态标签被 `//` 注释吞掉。本文描述模块设计意图和可见有效逻辑；正式仿真前必须恢复损坏行。
2. 端口名中的 `clk_192M` 是历史命名，实际频率应沿顶层时钟连接确认，不能仅凭端口名断言。
3. `FFT_*` 等信号名不等于已经证明执行标准 FFT/IFFT；应继续追踪 `Combine_Control_and_Data` 内的实际变换模块。

## 1.1 TX_Trigger

### 实现功能

收到一次无线帧触发后，在 `TX_active=1` 时启动一个完整无线帧的 OFDM 符号触发序列，共输出140个 `symbol_start_trigger`。

### 实现原理解读

```text
radio_frame_trigger
 → 上升沿检测
 → enable脉冲
 → TX_Symbol_Start_Generator锁存“正在发一帧”
 → 符号周期计数器按sym_dur循环
 → 每次回卷输出一个symbol_start
 → 符号数计到140后finish清除锁存
```

`feedback_en_7_U16_bmp` 每完成一个符号切换到下一个槽，形成7符号循环。当前七个槽最终只使用两个输入值，而 `FEEDBACK_1`、`FEEDBACK_2` 都配置为17443，因此当前没有区分一个 slot 内不同 CP 长度的符号节拍。

### 子模块关系图

```text
TX_Trigger
├─ delay_1(N=1)：上升沿检测
└─ TX_Symbol_Start_Generator
   ├─ sync_latch_another：一帧期间保持enable
   ├─ feedback_en_7_U16_bmp：7槽符号时长选择
   ├─ Mod_N_Indexer：符号内周期计数
   ├─ Mod_N_Indexer：无线帧内符号计数
   └─ delay_1×2：缓存finish和symbol_start
```

### 关键代码解读

```verilog
trigger_in_rising = trigger_in & (~trigger_in_delay1);
enable <= trigger_in_rising & TX_active;
```

符号总数来自：

```verilog
.N(`OFDM_SYMBOL_PER_RADIO10)   // 140
```

无线帧、子帧、slot、符号关系：

```text
1 radio frame = 10 subframe
1 subframe    = 2 slot
1 slot        = 7 OFDM symbol
总计          = 10×2×7 = 140 symbol
```

`Program_Const.vh` 当前写的是：

```verilog
FEEDBACK_1 = 17443
FEEDBACK_2 = 17443
```

宏后旧注释给出的两条算式分别会得到17464和17336，都不等于17443。因此只能确认17443是当前实际符号触发间隔，不能再把旧注释当作严格推导。若基带时钟为245.76 MHz：

```text
17443 cycle ≈ 70.976 μs
17443×140 = 2,442,020 cycle ≈ 9.9369 ms
```

剩余约63.1 μs是否对应保护间隔、流水余量或其他系统安排，仍需结合 RRH/CP 插入波形验证。

### 讨论问题

1. **17443是子载波数量吗？** 不是。它的单位是时钟周期，控制时域中的符号触发间隔。
2. **子载波和符号分别属于什么维度？** 子载波索引是频域坐标；OFDM符号索引是时域坐标。
3. **为什么代码注释说“每子帧140个符号”不对？** 140是一个10 ms无线帧的符号数，一个子帧只有14个符号。
4. **CP在哪里？** 当前模块只产生符号边界，CP样点的实际插入应在后续 RRH 链继续核实。

## 1.2 TTI_Handing_Top

### 实现功能

将 `TX_Trigger` 的符号脉冲变成无线帧、子帧、OFDM符号、子载波四级坐标，并在帧/子帧/符号边界输出控制脉冲。

### 实现原理解读

```text
无线帧计数 counter：每完成一帧加1
子帧索引 Sub：0..9
符号索引 Sym：0..13（子帧内）
子载波索引 Carrier：0..3599（符号内）
```

每来一个 `iSymbolp`，`Index_Generator` 更新 `Sym/Sub`。同时 `PULSE_TO_STROBE_U16(N=3600)` 把符号脉冲展宽为3600拍，在展宽期间给出 `oCarindex=0..3599`。

### 子模块关系图

```text
TTI_Handing_Top
├─ Index_Generator
│  ├─ sync_latch
│  ├─ PULSE_TO_STROBE_U16(N=3600)
│  └─ delay_n×6
├─ delay_en：帧起始锁存MCS_array
├─ delay_1(N=2)：PDCCH trigger
└─ delay_n(N=2)：输出统一对齐
```

### 关键代码解读

`Index_Generator` 使用五个条件拼成 case 选择码：

```text
{有无符号脉冲,
 是否非子帧末符号,
 是否子帧首符号,
 是否非末子帧,
 是否首子帧}
```

主要状态转移：

```text
Sym<13              → Sym++
Sym=13且Sub<9       → Sym=0，Sub++，start_of_subframe=1
Sym=13且Sub=9       → Sym=0，Sub=0，start_of_radio_frame=1
```

CM message布局：

```text
[111:88]  无线帧计数器
[87:84]   固定子帧数10
[83:24]   MCS_array
[23:0]    保留
```

### 讨论问题

1. **为什么 `Index_Generator` 可读性差？** 代码保留明显的 LabVIEW FPGA 自动翻译风格：多条件 case、功能块封装和大量中间延迟，没有按人工 RTL 常见的“显式四级计数器”书写。
2. **四级计数是否都是同一时钟每拍加1？** 不是。子载波在 `oCarrier` 有效期间逐拍加；符号只在符号脉冲到来时加；子帧只在符号13结束时加；无线帧只在子帧9结束时加。
3. **3600和4096是什么关系？** 当前工程定义3600个有效子载波；4096通常是后续变换长度。有效频域资源不等于变换总点数，中间还要留DC和保护带。

## 1.3 TX_PDSCH_Configuration

### 实现功能

在每个子帧开始时，根据 `MCS_control` 和子帧号选择调制方式、传输块大小，并生成启动 MAC 与 PXSCH 编码的触发。

### 实现原理解读

每个子帧产生三次独立的 PDSCH transaction：

```text
第1次：基础配置脉冲
第2次：基础脉冲再延迟 ENCODE_SEG=80130 cycle
第3次：基础脉冲再延迟 ENCODE_SEG2=160260 cycle
最终：三路脉冲 OR 后寄存输出
```

三次触发都会送到 `MAC_TX.MAC_trigger` 和 `PXSCH_TX_Bit_Processing_Top.PXSCH_transmitter_trigger`，因此不是 Turbo 码块分段触发，而是三个独立 TB/MAC 帧的启动。

这与资源规模闭合：每次 transaction 生成一组10800或15600调制符号；三次分别得到32400或46800符号，对应3600个子载波上的9或13个 PDSCH OFDM符号。

### 子模块关系图

```text
TX_PDSCH_Configuration
├─ CM_to_PDSCH_Encoder
│  └─ MCS_to_TBS
├─ Trigger_delay(80130)
├─ Trigger_delay(160260)
└─ delay_n：调制/TBS/符号与帧触发对齐
```

### 关键代码解读

三路触发：

```verilog
PDSCH_transmitter_trigger <=
    temp | temp_delay_80130 | temp_delay_160260;
```

TBS查表，每次触发对应一个TB：

| `MCS_control` | 子帧0 TBS(bit) | 其他子帧 TBS(bit) | 调制方式 |
|---:|---:|---:|---|
| 0 | 7224 | 10296 | QPSK |
| 1 | 8760 | 15840 | QPSK |
| 2 | 15840 | 29296 | QPSK |
| 3 | 8760 | 30576 | 子帧0 QPSK；其他16QAM |

实际调制选择：

```verilog
modulation <= ((subframe_index == 0) || (MCS_control != 3))
            ? QPSK : QAM16;
```

### 讨论问题

1. **为什么有三个MAC帧？** 因为当前资源网格每个子帧需要处理三段PDSCH数据；三次触发分别启动三次完整成帧和编码，而不是一个TB的三个码块。
2. **MCS如何选择？** 当前实际只由2位 `MCS_control` 和 `subframe_index` 决定。`MCS_array`、`layer_ID` 虽然进入端口，但 `CM_to_PDSCH_Encoder` 当前没有用它们参与计算。
3. **TBS=15840的非0子帧存在什么问题？** 参数表只按TBS查速率匹配参数。当前该TBS每次只产生10800个QPSK符号，三次共32400，少于非0子帧计划的46800，属于待修配置组合。
4. **三次触发能否重叠？** 间隔80130拍。是否对所有TBS都留有足够流水空间，应通过三次 transaction 的 ready/valid 波形确认。

## 1.4 Buffer_UDP_Data

### 实现功能

用异步 FIFO 将 UDP 时钟域的8位字节流搬运到基带时钟域，并向 `MAC_FIFO` 提供可读字节数。

### 实现原理解读

```text
UDP写侧：clk_udp，data_udp_valid，data_udp[7:0]
FIFO跨域
基带读侧：clk_baseband，ask_for_data，data_out[7:0]
```

读侧由下游拉取：

```verilog
data_valid = fifo_valid & ask_for_data;
rd_en      = ask_for_data;
```

### 子模块关系图

```text
Buffer_UDP_Data
└─ FIFO_Buffer_UDP（Xilinx异步FIFO）
   ├─ 写端：UDP时钟域
   ├─ 读端：基带时钟域
   └─ rd_data_count：FIFO可读字节数
```

### 关键代码解读

关键接口：

```verilog
.wr_en(data_udp_valid)
.rd_en(ask_for_data)
.valid(fifo_valid)
.rd_data_count(fifo_read_count)
```

`MAC_FIFO` 根据：

```text
min(TB_size/8 - 4, FIFO当前可读字节数)
```

决定本次帧装入多少payload，避免读空。

### 讨论问题

1. **是否形成了端到端反压？** 没有。FIFO的 `full` 只连到模块内部 wire，没有输出给 UDP 写端；`wr_en` 仍直接等于 `data_udp_valid`。FIFO满时后续写入可能被IP拒绝，当前接口无法通知上游重发。
2. **FWFT有什么作用？** 数据在FIFO非空时预先出现在 `dout`，读使能用于消费当前字，便于下游按需拉取。
3. **为什么需要大FIFO？** 主要解决异步跨时钟和UDP突发速率与基带消费速率不一致；具体深度以FIFO IP配置为准。

## 1.5 MAC_TX / MAC_FIFO

### 实现功能

`MAC_TX` 是薄封装；核心 `MAC_FIFO` 把 UDP 字节流组成固定 `TB_size` 的串行bit流：

```text
32-bit payload长度头
+ N字节payload
+ 若干0 padding
= TB_size bit
```

它是工程自定义成帧器，不是完整 LTE MAC。

### 实现原理解读

六状态机：

```text
IDLE
 → COMPUTE_REMAINING_CONFIGS
 → SEND_HEADER
 → READ_FIFO ⇄ SEND_PAYLOAD
 → SEND_PADDING
 → IDLE
```

处理过程：

1. trigger 到来后锁存 `TB_size` 和 FIFO 可读字节数；
2. payload字节数取“本帧容量”和“FIFO实际数据量”的较小值；
3. 先按LSB first发送32位payload字节数；
4. 每读一个字节，用8个有效拍按LSB first发送；
5. 用0补齐到固定 `TB_size`。

### 子模块关系图

```text
MAC_TX
└─ MAC_FIFO
   ├─ delay_n：trigger/TB_size对齐
   ├─ 六状态机：组帧和串行化
   └─ FIFO_MAC_handshake：1-bit输出缓冲
```

### 关键代码解读

单位换算：

```verilog
output_length_div8        = TB_size >> 3;       // byte
output_length_div8_minus4 = (TB_size >> 3) - 4; // byte
output_length_minus32     = TB_size - 32;       // bit
```

帧头和payload：

```verilog
SEND_HEADER : bit_out_reg <= remaining_FIFO_elements[bit_counter];
SEND_PAYLOAD: bit_out_reg <= FIFO_data[bit_counter];
SEND_PADDING: bit_out_reg <= 1'b0;
```

例：`TB_size=288 bit`、FIFO有20 byte：

```text
header  = 32 bit
payload = 20×8 = 160 bit
padding = 288-32-160 = 96 bit
```

### 讨论问题

1. **为什么逐bit输出？** 当前 CRC/Turbo/速率匹配接口按1 bit/cycle设计，所以这里提前完成字节到bit串行化。这是当前RTL架构选择，不是Turbo算法理论上绝对不能并行。
2. **是不是标准LTE MAC？** 不是。它没有逻辑信道复用、标准MAC PDU子头、调度和HARQ，只实现长度头、payload和padding。
3. **“循环20次”是什么意思？** 示例中有20个payload字节；每轮 `READ_FIFO` 读1 byte，再用8个 `SEND_PAYLOAD` 有效拍发完。
4. **`minus4`和`minus32`为什么都存在？** 前者以byte表示32-bit帧头，后者以bit表示同一帧头，分别服务字节计数和bit计数。
5. **反压到哪里为止？** `ready_for_output` 能暂停本模块向编码器输出，也能停止继续读UDP FIFO；但不能让已满的UDP FIFO通知网络写端停止。
6. **当前源码风险？** `MAC_FIFO.v` 多处换行被乱码注释吞掉，`last_sample_out` 的有效赋值也被注释。仿真前必须先恢复源文件。

## 1.6 PXSCH_TX_Bit_Processing_Top

### 实现功能

把 `MAC_TX` 输出的一个TB串行bit流转换为PDSCH复数调制符号，同时把原始OFDM符号触发延迟一个固定安全窗口后交给资源映射模块。

### 实现原理解读

数据路径：

```text
TB payload bit
 → 参数查表
 → TB/CB CRC与码块分段
 → Turbo编码
 → 速率匹配
 → Qm个bit组成调制字
 → Gold序列加扰
 → QPSK/16QAM映射
 → Q2.14 I/Q
```

控制路径：

```text
PDSCH trigger
 → 锁存modulation/subframe/TBS/RE数量
 → 延迟1拍启动编码器和扰码初值生成

OFDM symbol trigger
 → Trigger_Delay_10x(delay=20000)
 → 后续资源映射时间线
```

#### PXSCH_Parameter_Computation

当前不是按协议公式实时推导，而是仅根据 `transport_block_size` 查表，输出码块数C、K、D、E、k0、交织行数等参数，再统一延迟约97/98拍送出。

`number_of_REs`、`modulation`、`redundancy_version_index` 虽然进入 `PXSCH_Channel_Encoder` 端口，但没有参与当前参数表选择。RV宏固定为0。

#### CRC_Add_and_CB_Segmentation

执行TB CRC、码块分段和必要的CB CRC。一次外部PDSCH trigger会在内部状态机中处理本TB的全部码块，不需要“每个码块再触发一次”。

#### Turbo_Encoder_Top

```text
原始序列 → RSC编码器1 → systematic + parity1
原始序列 → QPP交织器 → RSC编码器2 → parity2
```

当前RTL使用两个 `Turbo_Encoder` 实例，一个处理原始顺序，一个处理交织顺序。QPP交织地址形式为：

```text
π(i) = (f1·i + f2·i²) mod K
```

每个RSC在码块结束时还要输出栅格终止信息，使编码器状态回到已知状态。LTE Turbo母码率约为1/3，后续通过速率匹配适配实际资源。

#### TX_Rate_Matcher

三个Turbo子流先进行32列子块交织，再写入逻辑循环缓冲区。发送端从 `k0` 开始按地址：

```text
addr(j) = (k0+j) mod Ncb，j=0..E-1
```

连续读取E个bit：

```text
E<Ncb：部分地址未读，形成打孔
E=Ncb：每个地址读取一次
E>Ncb：地址回卷，形成重复
```

被打孔的bit没有单独删除信号，只是写入RAM后没有被读出；该页后续会被新码块覆盖。接收端在对应位置填LLR=0，表示“未知”而不是硬bit 0。

TB=30576的当前参数：

```text
C=5
K=6144
D=K+4=6148
Ncb=3D=18444
E=12480 bit/code-block
k0=384
G=C×E=62400 coded bit
16QAM符号数=62400/4=15600
每码块打孔数=18444-12480=5964
```

#### PXSCH_Deserializer_Scrambler_Modulator

输入是串行coded bit，输出是复数调制符号：

```text
bit/valid延迟33拍等待扰码初值
 → 每Qm个bit组成一个调制字
 → Gold加扰
 → 星座映射
```

QPSK每2个有效bit输出一个符号，16QAM每4个有效bit输出一个符号。当前底层 `Modulation.v` 只例化QPSK和16QAM；上层虽然定义64QAM枚举和6-bit聚合，但选择64QAM时没有有效调制输出。

Gold序列：

```text
x1(n+31)=x1(n+3)⊕x1(n)
x2(n+31)=x2(n+3)⊕x2(n+2)⊕x2(n+1)⊕x2(n)
c(n)=x1(n+1600)⊕x2(n+1600)
```

当前PDSCH初始化相当于码字索引 `q=0`：

```text
c_init = cell_ID | (RNTI<<14) | (subframe_index<<9)
```

`x1_initial_value=0x5E485840` 是预先跳过1600步后的状态；`x2` 用ROM中的线性变换掩码从 `c_init` 计算。发送端核心：

```verilog
xor_result = data_in ^ x1_state[7:0] ^ x2_state[7:0];
```

每个有效符号使用状态低Qm位，然后把x1/x2状态推进Qm步。接收端对硬bit可再次异或同一Gold序列；对LLR则在Gold bit为1时取相反数。

### 子模块关系图

```text
PXSCH_TX_Bit_Processing_Top
├─ PXSCH_Channel_Encoder
│  ├─ PXSCH_Parameter_Computation
│  ├─ CRC_Add_and_CB_Segmentation
│  ├─ Turbo_Encoder_Top
│  │  ├─ QPP Interleaver
│  │  └─ 2×Turbo_Encoder(RSC)
│  └─ TX_Rate_Matcher
│     ├─ TX_Rate_Matcher_FSM
│     ├─ 子块交织地址生成
│     ├─ 双页循环缓冲区
│     └─ 线性读地址生成
├─ PXSCH_Deserializer_Scrambler_Modulator
│  ├─ Scrambler_Cinit_to_X1_X2
│  ├─ PXSCH_Deserializer_Bit_to_Symbol
│  ├─ PXSCH_Scrambler
│  └─ LTE_Modulation
├─ Trigger_Delay_10x
└─ 输出I/Q与valid统一打拍
```

### 关键代码解读

配置先锁存、trigger后到：

```text
T0：锁存modulation、subframe、TBS、RE number
T1：延迟后的trigger启动Channel Encoder和扰码初值计算
```

总速率匹配输出必须按：

```text
G = ΣE_r
调制符号数 = G/Qm
```

不能把“每码块E”和“全部码块总输出G”再次相乘。

`ENCODE_SAFE_VALUE=20000` 只连接 `Trigger_Delay_10x.delay`，表示把OFDM时间线推迟20000个基带时钟周期，不等于整个信道编码固定延时。

`Trigger_Delay_10x` 名字写10路，实际包含14个并行槽。实现还存在：

```text
变量名delay_minus_2，实际计算delay-3
delay<3时无符号下溢
下降沿检测else缺少begin/end，复位只完整控制falling[0]
```

### 讨论问题

1. **整个模块是不是固定37拍？** 不是。bit先固定延迟33拍，但bit-to-symbol需要等Qm个有效bit，LTE调制当前为4拍，顶层还会再打拍。输入和输出粒度不同，精确首/末符号延时必须用波形测量。
2. **速率匹配“删掉”的bit怎么办？** 发端只是不读；收端对应位置使用LLR=0。若E>Ncb发生重复，标准接收应合并同地址LLR；当前F0接收循环缓冲区没有实现累加，只会覆盖，损失重复增益。
3. **是否实现HARQ增量冗余？** 当前RV固定0、k0由TBS表给出，接收端也没有跨传输软合并，不能称为完整HARQ IR。
4. **Gold加扰是不是加密？** 不是。它可重复、初值可由协议参数得到，用于白化和区分数据流，不提供机密性。
5. **加扰和主动干扰有什么区别？** 加扰在调制前做 `b_tilde=b⊕c`，收端可逆且不增加射频能量；主动干扰是在信道叠加额外波形 `y=h·s+j+n`，通常未知且不可直接异或消除。
6. **当前64QAM能用吗？** 不能。只有枚举和6-bit聚合，底层没有64QAM调制实例。
7. **最小仿真观察点是什么？** 记录PDSCH trigger、ready、MAC bit valid、Turbo/Rate Matcher输出valid、调制valid、symbol valid，以及每个OFDM trigger输入输出的周期差。

## 1.7 PDCCH_Transmitter_Top

### 实现功能

把 `TTI_Handing_Top` 生成的控制消息转换为PDCCH调制符号，供后续资源映射与PDSCH复用。

### 实现原理解读

```text
CM_message
 → CM_to_Stream：控制消息串行化
 → PDCCH_TX_Bit_Processing：控制信道编码、加扰、调制
 → PDCCH I/Q symbol
```

### 子模块关系图

```text
PDCCH_Transmitter_Top
├─ delay_1：PDCCH trigger对齐
├─ CM_to_Stream
└─ PDCCH_TX_Bit_Processing
   └─ 复用PXSCH_Deserializer_Scrambler_Modulator完成后段加扰调制
```

### 关键代码解读

顶层先把 `PDCCH_trigger` 延迟1拍，再让 `CM_to_Stream` 输出控制bit，随后进入PDCCH bit processing。当前文档只完成调用关系确认，编码细节留待精读。

### 讨论问题

1. PDCCH与PDSCH最终在 `Combine_Control_and_Data` 中按资源网格位置选择，不是在bit级直接拼接。
2. 后续需核实控制消息实际长度、编码方式、资源位置和当前是否完整遵循LTE PDCCH格式。

## 1.8 Combine_Control_and_Data

### 实现功能

缓存PDCCH/PDSCH符号，生成参考信号，按照当前OFDM符号和子载波位置完成频域资源映射，再执行DC插入和WFRFT发送变换。

### 实现原理解读

```text
PDCCH/PDSCH I/Q
 → 各自FIFO缓存
 → TX_Resource_Mapper_Top决定当前位置的符号类型
 → RS_need_code_Gen生成参考信号
 → 选择PDCCH/PDSCH/RS/0
 → DC_Insertion
 → WFRFT_TX
 → FFT_*命名的复数输出
```

### 子模块关系图

```text
Combine_Control_and_Data
├─ WFRFT_Alpha_Indicate
├─ TX_Resource_Mapper_Top
├─ RS_need_code_Gen
├─ FIFO_for_PDCCH_symbol
├─ FIFO_for_PDSCH_symbol
├─ DC_Insertion
└─ WFRFT_TX
```

### 关键代码解读

`TX_Resource_Mapper_Top` 负责回答“当前资源粒子放什么”；两个FIFO负责回答“对应数据符号是什么”；参考信号发生器提供RS；随后插入DC并进入变换模块。

3600表示有效子载波数量，不等于变换点数。`LEFT_SPACE`、`RIGHT_SPACE`、DC位置和4096点内部网格的精确关系需在本模块精读时建立地址表。

### 讨论问题

1. 信号名 `FFT_valid/FFT_symbol` 是接口历史命名，不能单凭名字断言方向；当前实际最后一级是 `WFRFT_TX`。
2. `using_ifft` 被固定延迟10000拍，说明变换模式还有一条独立控制时序，需要结合 `alpha` 和WFRFT内部状态核实。
3. 本模块是下一阶段最重要的时频连接点，应重点建立“符号索引×子载波索引→PDCCH/PDSCH/RS/DC/保护带”的表。

## 1.9 Invalidate_Layer_Streams

### 实现功能

当某个layer未激活时，将该layer的复数数据强制为0，但保留原来的 `valid` 节拍，使后续MIMO模块仍能按完整网格计数。

### 实现原理解读

```text
layer_active=1：数据透传，valid透传
layer_active=0：I/Q清零，valid仍透传
```

### 子模块关系图

```text
Combine_Control_and_Data输出
 → Invalidate_Layer_Streams
    ├─ active：保留I/Q
    └─ inactive：I/Q=0
 → TX_BIT_FIFO_Exchange
```

### 关键代码解读

```verilog
valid_out <= valid_in;

if(layer_active) begin
    symbol_out_real <= symbol_in_real;
    symbol_out_imag <= symbol_in_imag;
end else begin
    symbol_out_real <= 0;
    symbol_out_imag <= 0;
end
```

### 讨论问题

1. **为什么不把valid也清零？** 后续MIMO处理按valid统计行、子载波和RB位置；清除valid会缩短网格并让其他layer错位。零数据表示该layer在当前位置不贡献能量。
2. 数据寄存器没有显式复位分支，复位后到首个有效时钟前仿真可能看到X；是否需要补复位取决于下游是否严格受valid门控。

# 2. TX_BIT_FIFO_Exchange

## 2.0 TX_BIT_FIFO_Exchange 顶层

### 实现功能

把本 FPGA 三条本地 layer 的16位I、16位Q复数流缓存、重排并打包为64位交换数据，同时输出目标 `FPGA_ID`，交给 `FIFO_Manager/Aurora` 做跨板 MIMO 数据汇聚。

### 实现原理解读

每条 layer 的一个复数样点先组成32位：

```text
{imag[15:0], real[15:0]}
```

三条layer使用三个32位FIFO平滑时序，控制逻辑再按12拍分段选择不同layer/直通数据，最后 `TX_BIT_Data_Processing` 把32位流进一步组成64位交换字。

### 子模块关系图

```text
Layer0/1/2 I/Q
 → 3×FIFO_for_TX_BIT_Buffer
 → 时分选择与目标FPGA编号
 → 32-bit oData
 → TX_BIT_Data_Processing
 → 64-bit data + FPGA_ID
 → FIFO_Manager/Aurora
```

### 关键代码解读

```verilog
FIFO0.din = {iL0I,iL0R};
FIFO1.din = {iL1I,iL1R};
FIFO2.din = {iL2I,iL2R};

oData <= oD0 | oD1 | oD2 | oD3;
```

未选中的分支被valid门控为0，因此可以用按位OR完成多路选择。目标 FPGA 顺序由内部计数器生成，当前可见序列为2、1、3、0。

### 讨论问题

1. 该模块控制代码大量使用 `cnt0/cntt0/long_t*` 和脉冲展宽器，可读性较低；后续精读应画出12拍输入和64位输出的逐拍表。
2. 三个内部FIFO的 `full/empty` 没有进入显式保护，需检查上游固定节拍是否保证不会溢出。
3. `FGPA_ID` 是源码端口拼写，语义仍是 `FPGA_ID`。

# 3. TX_MIMO_Processor

## 3.0 TX_MIMO_Processor 顶层

### 实现功能

从跨板汇聚后的128位 payload FIFO 读取12层数据，按当前“直接预编码”规则扩展成32个发射天线行，再拆分并打包到两路64位 MIMO-to-RRH FIFO。

### 实现原理解读

```text
128-bit layer payload
 → 按RB读取12个连续行
 → TX_Precoding_Direct扩展为32个天线行
 → Submatrix_Splitter分两组
 → 2×MIMO_TX_Pack_Data
 → 两路64-bit FIFO
```

这里的 `TX_Precoding_Direct` 没有执行信道相关矩阵乘法。它在检测到输入valid上升沿后产生长度为 `NUMBER_ANTENNA=32` 的输出窗口：

```text
index 0..11  ：输出当前12行
index 12..23 ：输出延迟12拍的同一组12行
index 24..31 ：输出延迟24拍后的前8行
```

等价于把12层行序列重复成32个天线行，是固定直接映射，不是常见的 `x=W·s` 自适应/码本预编码。

### 子模块关系图

```text
TX_MIMO_Processor
├─ MIMO_TX_Trigger：按RB和FIFO余量产生读取节拍
├─ MIMO_Processor_Payload_Reader：读出12行子矩阵
├─ TX_Precoding_Direct：12层直接重复到32天线行
├─ Submatrix_Splitter：拆成前/后天线组
└─ 2×MIMO_TX_Pack_Data：打包64位FIFO字
```

### 关键代码解读

Payload Reader 输出模式的源码注释为：

```text
连续12拍输出一个子矩阵，随后空32拍
```

直接预编码选择：

```verilog
if(index < 12)       data_out <= data_delay1;
else if(index > 23)  data_out <= data_delay25;
else                 data_out <= data_delay13;
```

两路打包分别对应前16根和后16根天线的数据组织。

### 讨论问题

1. **当前是否真正做了MIMO预编码？** 只做固定直接映射/重复，没有看到信道矩阵或码本权重乘法。
2. **为什么输入12行、输出32行？** 系统定义12层、32发射天线；当前模块用重复映射补足32个天线行。
3. **128位一拍代表什么？** 当前注释表示4列复数数据的组合，精确位域和行列方向应在 `MIMO_Processor_Payload_Reader`、`Submatrix_Splitter` 精读时画图确认。

# 4. TX_RRH_FIFO_Exchange

## 4.0 TX_RRH_FIFO_Exchange 顶层

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

# 5. TX_RRH_Processor

## 5.0 TX_RRH_Processor 顶层

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

# 6. Over_Sample_Group

## 6.0 Over_Sample_Group 顶层

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

---

学习顺序建议：

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

待验证清单：

- [ ] 恢复被乱码注释吞掉的 RTL 连接并通过语法检查；
- [ ] 用波形测量17443、17543和 `Trigger_Delay_10x` 的真实周期差；
- [ ] 核对 `Combine_Control_and_Data` 的3600有效子载波到4096点内部网格映射；
- [ ] 验证每子帧三次PDSCH transaction与9/13个PDSCH符号的完整闭合；
- [ ] 精读 `TX_BIT_FIFO_Exchange` 的12拍调度和64位数据格式；
- [ ] 精读 `TX_RRH_Chain` 内部的变换、CP插入和天线时域输出；
- [ ] 验证 `Over_Sample_Group` 的实际上采样倍率与定点缩放。

---

*最后更新：2026-08-10*
