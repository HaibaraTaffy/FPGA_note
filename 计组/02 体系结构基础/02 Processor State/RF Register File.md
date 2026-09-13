即 寄存器堆 是 **一组寄存器 + 统一的寻址与读写接口**

## 地址
---
一般 RF的接口中 会出现
- Read Address : 读地址
- Write Address : 写地址
**这里的地址 并非内存地址 只是 RF的内部寄存器编号**
**于是 通常也被称为 Register Index 寄存器索引 或 Register Number 寄存器编号**

## 接口
---
先看一个只有一个读端口、一个写端口的 RF。
```
               ┌──────────────────┐
Read Address ─→│                  │
               │  Register File   ├──→ Read Data
Write Address─→│                  │
Write Data   ─→│                  │
Write Enable ─→│                  │
               └──────────────────┘
```
它至少有：
读接口
```
Read Address
Read Data
```
写接口
```
Write Address
Write Data
Write Enable
```

## 读操作
---
即 **根据 Read Address 从多个寄存器中选择一个数据输出到 Read Data线上**
本质上包含一个 **多选一的数据选择网络**

读不一定是同步读 视具体实现 可能采用
- 组合读
- 同步读
- 特定工艺


## 写操作
---
即 **根据 Write Address的选择 和 Write Enable 的控制 将Write Data上的数据 写入Address对应的寄存器中**
Write Address 决定写谁 Write Data 决定写什么
可以用 `Decoder + Write Enable` 来理解

## 2R1W
---
即 两读一写
- 2个**独立**读端口
- 1个写端口
```
                 Register File
        ┌──────────────────────────┐
Read Addr 1 ─→                     ├──→ Read Data 1
Read Addr 2 ─→                     ├──→ Read Data 2
        │                          │
Write Addr  ─→                     │
Write Data  ─→                     │
Write Enable─→                     │
        └──────────────────────────┘
```
很多指令 会需要两个源操作数 需要同时读取两个寄存器 所以需要**两个读端口**
运算完成后 结果通常需要写回 于是需要**一个写端口**

## Port
---
即 一组完成一个存储访问所需的接口信号
例如 一个Write Port可能包含
```
Write Address
Write Data
Write Enable
Clock
```

由此可以看到 两个独立读端口 可以在同一时刻 读取不同或相同寄存器的值

## 读写"冲突"
---
当读写对应同一地址时 
Read Data 到底是：
- 旧值；
- 新值；
- 还是通过旁路逻辑直接拿到新值；

具体实现取决于具体微架构和RF实现 需要进行明确规定

## Write Decoder
---
译码器 会将**二进制地址 转换成 独热码 达到 只选择其中一路的效果**

## 容量 端口数 访问速度的平衡
---
每增加一套独立的端口 那么每个Register的数据都要能够送到更多选择网络 会增加
- MUX 数量；
- 互连数量；
- 布线压力；
- 面积；
- 功耗；
- 路径延迟

也不能把RF容量做的太大 不然增加地址位宽 无论采用什么具体电路结构 都会 :
- 选择网络更复杂；
- 数据线更长；
- 译码更复杂；
- 面积和功耗上升；
- 高频实现更困难。

**容量大 端口多 访问快 这三件事很难同时无限满足**

## 物理实现
---
Register File 不一定是 SRAM 也不一定是FF
**Register FIle 首先是一种功能组织结构**

## RF与Instruction
---
假设 Instruction 里有字段 :
```
rs1
rs2
rd
```
- rs - Source Register - 源寄存器
- rd - destination Register - 目的寄存器

例如
```
rs1 = 1
rs2 = 2
rd  = 3
```
则RF可以收到
```
Read Address 1 = 1
Read Address 2 = 2
Write Address = 3
```
**Instruction 中的 Register 字段 本质上就是RF的索引信息 而不是操作数本身**

