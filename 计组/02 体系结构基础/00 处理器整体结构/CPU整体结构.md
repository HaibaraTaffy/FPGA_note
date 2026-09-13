```
CPU
│
├─ Processor State 处理器状态
│
├─ Datapath 数据通路
│		└─ 数据的存储、选择、传递、计算
│
└─ Control 控制逻辑/控制通路
		└─ 指令译码与控制信号生成/传播
```

目前的功能骨架
```
                  Instruction
                       │
                       ▼
                ┌─────────────┐
                │   Decode    │
                └──────┬──────┘
                       │
                    Control
                       │
          ┌────────────┼────────────┐
          │            │            │
          ▼            ▼            ▼

 PC → Instruction → Register File → ALU → Memory
                         ▲            │
                         └────────────┘
                            Write Back
```

## Processor State
---
即 CPU当前保存的状态信息
```
Register
Register File
PC
```
都属于状态存储的相关硬件

但是 Processor State 不等于 Architectural State 体系结构状态
即 **CPU保存的状态很多 但并不是所有状态都由ISA定义**

## Datapath
---
数据通路 即 **CPU中负责保存 传递 选择和运算数据的硬件路径**
通常
```
Datapath
│
├─ State Storage 状态存储
│  ├─ PC 程序计数器
│  └─ Register File 寄存器组
│
├─ Computation 计算
│  ├─ ALU 算术逻辑单元
│  ├─ Adder 加法器
│  ├─ Comparator 比较器
│  └─ Shifter 移位器
│
├─ Selection 选择
│  └─ MUX 多路选择器
│
└─ Data Transfer 数据传输
   ├─ Bus 总线
   └─ Internal Connections 内部联系
```

## Control
---
即 Control Logic 控制逻辑 
**根据当前 Instruction 和 CPU 状态 产生控制硬件行为所需的 Control Signal 即控制信号**
Control 会向 Datapath 中很多部件发出控制信号

**但是Instruction 不等于 Control Signal**
`Instruction -> Decode -> Control Signals`

很多的控制 都是通过 **MUX** 来进行实现

## 最基本的同步结构
---
即**状态机**
从最底层的同步数字系统视角：
```
       State
         │
         ▼
Combinational Logic
         │
         ▼
     Next State
         │
         ▼
      Register
```
Clock Edge 到来：
`Current State -> Next State`
CPU 本质上仍然遵循这个模型

## Control Path
---
控制通路
将CPU内部负责控制的信息传播也看成一套路径 则可以形成Control Path

**不必要过分区分 Data Path 和 Control Path**

## CPU内部架构总图
---
```
CPU
│
├─ Processor State
│  ├─ Architectural State
│  └─ Microarchitectural State
│
├─ Datapath
│  ├─ Storage
│  │  ├─ Register File
│  │  └─ PC
│  │
│  ├─ Computation
│  │  ├─ ALU
│  │  ├─ Adder
│  │  ├─ Comparator
│  │  └─ Shifter
│  │
│  └─ Selection / Transfer
│     ├─ MUX
│     └─ Internal Connections
│
└─ Control
   ├─ Instruction Decode
   ├─ ALU Control
   ├─ MUX Select
   ├─ Register Write Enable
   ├─ Memory Control
   └─ PC Control
```