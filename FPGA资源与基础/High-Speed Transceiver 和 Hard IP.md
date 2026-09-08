随着数据率提高 普通IO已经无法承担这么高速的数据传输了
会出现
```
- 极高速串行化/解串
- Clock Recovery
- 精确高速采样
- 信号均衡
- 发送端驱动
- 接收端模拟前端
- 编码/解码
- 高速差分电气接口
```
一系列问题
于是FPGA内部出现了专门的
`High-Speed Serial Transceiver(高速串行收发器)`
属于FPGA的专用硬件资源

## Transceiver
---
其实是 `Transmitter + Receiver` 即发送器 + 接收器
本质上是一个**双向高速串行通信硬件模块**
使用场景
```
                 Transceiver

FPGA Fabric
     │
     │ Parallel Data
     ▼
┌──────────────────┐
│    Transmitter   │
│                  │
│    Serializer    │
│        ↓         │
└────────┬─────────┘
         │
         ▼
      TX Serial
         │
         │ 高速串行链路
         │
      RX Serial
         │
         ▼
┌────────┴─────────┐
│     Receiver     │
│                  │
│       CDR        │
│        ↓         │
│  Deserializer    │
└────────┬─────────┘
         │
         ▼
    Parallel Data
         │
         ▼
    FPGA Fabric
```
**FPGA内部 :低速 宽并行 <-> Transceiver <->外部 : 高速 窄串行**
通过Transceiver 可以在传输数据率保持相等的情况 降低时钟频率 达到
**把极高速串行世界转换成FPGA Fabric能够处理的并行世界**

## Transmitter
---
在发送(TX)方向 主要做:Parallel -> Serial
```
FPGA Fabric
     │
     │ Parallel Data
     ▼
Serializer
     │
     ▼
高速发送电路
     │
     ▼
TX_P / TX_N
```

## CDR
---
在高速串行链路中 通常没有独立的时钟 而是将时钟和数据通过`Serial Data`一起传输 在接收端 需要进行`CDR`
 即**Clock and Data Recovery** 时钟和数据恢复
RX端需要从接收到的高速串行信号的时序变化中 恢复出正确的采样时序
 例如接收到：
```
...1011001011101001...
```
接收器需要确定：
```
      ↓   ↓   ↓   ↓   ↓
... 1 | 0 | 1 | 1 | 0 | 0 ...
      ↑
   sampling point
```
也就是：确定Bit Boundary(边界) 和 采样时机
为了可以完成这两件事 串行信号本身必须具有适合恢复Timing的特性
设计者会设计 `Line Coding/Scrambling`等机制 满足协议所需的信号特性
例如我熟知的8b/10b 不熟知的Scrambler
**CDR会锁定到接受信号的时序特征 并产生用于恢复数据的采样Clock/Timing Reference**

## Lane
---
通常采用差分信号 构成
```
TX_P
TX_N

RX_P
RX_N
```
两根线共同构成一个 `Serial Lane`
一条Lane具有独立的RX TX两条差分对
一条Lane可以理解为 **一条独立的高速串行数据通道**

## 对比普通I/O SERDES
---
![[FPGA_IO_SERDES_与_Transceiver_对比.svg]]
Transceiver是一整套高速串行PHY硬件

## Equalization
---
高速信号经过传输 不会完美保持原状 尤其容易受到通道损耗影响 
所以高级Transceiver通常会有 `Equalization(均衡器)` 用于补偿通道造成的一部分失真

## Reference Clock
---
需要一个外部的参考时钟 使得Transceiver内部可以产生所需的高速串行时序
Transceiver 内部通常有专用：
- PLL
- Clock Divider
- Clock Distribution
等资源

## Hard IP Soft IP IP Core
---
Hard IP : 已经物理实现于芯片中的专用功能硬件 例如
```
PCIe Hard Block
Memory Controller
Ethernet MAC
Processor Core
Transceiver
```
功能固定 但是性能和功耗效率高 且不占大量Fabric

Soft IP : 主要通过一些可编程资源实现(LUT FF...)
灵活 可配置 可移植性高 但是占用Fabric 且性能功耗不如Hard IP

IP Core : 是设计/功能封装概念
一个IP Core可能是纯Soft 也可能调用Hard Resource 也可能Hard + Soft混合

## 总览FPGA资源
---
```
                    FPGA
┌──────────────────────────────────────┐
│                                      │
│  Programmable Fabric                 │
│                                      │
│  LUT ─ FF ─ Carry ─ Routing          │
│                                      │
│  ┌──────┐        ┌──────┐            │
│  │ BRAM │        │ DSP  │            │
│  └──────┘        └──────┘            │
│                                      │
│  Clock Resources                     │
│  PLL/MMCM ── Clock Network           │
│                                      │
│  Ordinary I/O                        │
│  Buffer / DDR / Delay / SERDES       │
│                                      │
│  High-Speed Transceiver              │
│  TX / RX / SERDES / CDR / EQ / PLL   │
│                                      │
│  Optional Hard IP                    │
│  PCIe / CPU / MAC / Controller...    │
│                                      │
└──────────────────────────────────────┘
```

