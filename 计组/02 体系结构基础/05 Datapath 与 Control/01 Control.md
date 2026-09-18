**Datapath 本身只提供可能的路径
Control 决定选择哪条路径

架构图
```
Instruction
     │
     ▼
   Decode
     │
     ▼
Control Logic
     │
     ├──────── RegWrite ──────→ Register File
     │
     ├──────── ALUSrc ────────→ ALU Input MUX
     │
     ├──────── ALUControl ────→ ALU
     │
     ├──────── MemWrite ──────→ Data Memory
     │
     ├──────── ResultSrc ─────→ Write-Back MUX
     │
     └──────── Branch ────────┐
                              │
Compare Result ───────────────┤
                              ▼
                            PCSrc
                              │
                              ▼
                         Next-PC MUX
```

## Control
---
**由 Control Logic 产生 Control Signal**
在 CPU 中：
**Control Logic — 控制逻辑**
负责根据：
- 当前 Instruction
- 当前处理器状态
- 某些运算结果
产生：
**Control Signal — 控制信号** 本质上就是数字信号
控制信号决定：
- MUX 选择哪个输入；
- ALU 执行什么操作；
- Register File 是否写；
- Data Memory 是否写；
- PC 选择哪个 Next PC。

于是存在流程
```
Instruction
     │
     ▼
   Decode
     │
     ▼
Control Logic
     │
     ├─ RegWrite
     ├─ ALUSrc
     ├─ ALUControl
     ├─ MemWrite
     ├─ ResultSrc
     └─ PCSrc
```

## Decode
---
译码 即**根据 Instruction Encoding 判断当前是什么指令 以及它需要什么硬件行为**

以 `add` 为例
```
Instruction Bits
      │
      ├──── rs1 / rs2 / rd ───→ Datapath
      │
      └──── opcode / funct ────→ Decode
                                   │
                                   ▼
                              Control Signals
```

## RegWrite
---
即 Register Write Enable 寄存器写使能
决定 **当前 Instruction 是否允许写 Register File**

从 RF 看
```
Write Address 对应rd
Write Data 对应 Result Data
Write Enable 对应 RegWrite
```

## ALUSrc
---
即 ALU Source Select ALU 输入来源选择
例如:
```
RF[rs2] ─────────┐
                 ▼
                MUX ───→ ALU Operand B
                 ▲
                 │
Immediate ───────┘
```
ALUSrc 控制这个 MUX **且ALUSrc可以由微架构设计者自己设计控制信号编码**

## ALUControl
---
ALU控制信号 **告诉ALU 现在到底执行什么运算**
例如可能定义：
```
000 → ADD
001 → SUB
010 → AND
011 → OR
100 → XOR
101 → SLT
...
```
具体 bit 编码由微架构自己决定 RISC-V 并没有规定

## MemWrite
---
Memory Write Enable 存储器写使能
**决定 是否向 Data Memory 写数据**

## 没有 MemRead
---
我们的设计
Data Memory 读取端 **默认根据Address输出数据** 
于是只需要明确控制 Write Enable 不需要MemRead

## ResultSrc
---
Result Source Select 结果来源选择
**控制 最终写回 RF 的数据来自哪里**
例如:
```
ALU Result ────────┐
                   ▼
                  MUX ─→ RF Write Data
                   ▲
                   │
Memory Read Data ──┘
```
具体表示意思 也是可以自由定义的 甚至可以扩充到多bit

## PCSrc
---
PC Source Select PC来源选择
**控制Next PC 的来源**
例如
```
PC + 4 ───────────┐
                  ▼
                 MUX ─→ Next PC
                  ▲
                  │
Branch Target ────┘
```
**PCSrc 不能只由 Opcode决定**

一种典型的设计 就是
- Decode 产生 `Branch = 1` 表示当前是一条*条件分支指令*
- 与此同时 Comparator/ALU 得到 `Zero/Equal`
- 然后 `PCSrc = Branch & Equal`

**这是一个很好的 Control 不一定只由 Instruction 决定的例子**

## 初步搭建 Control Table
---
控制表
我们假设
```
ALUSrc:
0 = Register
1 = Immediate

ResultSrc:
0 = ALU
1 = Memory
```

|Instruction|RegWrite|ALUSrc|ALUControl|MemWrite|ResultSrc|Branch|
|---|--:|--:|---|--:|--:|--:|
|`add`|1|0|ADD|0|0|0|
|`sub`|1|0|SUB|0|0|0|
|`addi`|1|1|ADD|0|0|0|
|`lw`|1|1|ADD|0|1|0|
|`sw`|0|1|ADD|1|X|0|
|`beq`|0|0|SUB / Compare|0|X|1|
X为无关项

**Control Unit 实际上就是把 Instruction 特征映射成这组 Control Signals**

例如：
```
opcode = LOAD
```
可能得到：
```
RegWrite = 1
ALUSrc   = 1
MemWrite = 0
ResultSrc = Memory
ALUOp    = ADD
```

## Control Vector
---
控制向量 
假设我们有：
```
RegWrite
ALUSrc
MemWrite
ResultSrc
Branch
```
就可以把它们组合起来看成一组 ：控制向量
例如 `lw`：
```
RegWrite = 1
ALUSrc   = 1
MemWrite = 0
ResultSrc = 1
Branch   = 0
```
本质上就是：**当前 Instruction 对整条 Datapath 的配置**

## 分层译码
---
一般 我们会把 Control 进行分层 例如:
```
Instruction
    │
    ▼
Main Decoder
    │
    ├─ RegWrite
    ├─ MemWrite
    ├─ ALUSrc
    ├─ ResultSrc
    ├─ Branch
    │
    └─ ALUOp
          │
          ▼
      ALU Decoder
          │
          ▼
      ALUControl
```
可以分出不同颗粒度的指令

## Main Decoder
---
主译码器
主要根据 `Opcode` 判断 : **当前属于哪一大类 Instruction**
例如 : 
```
Register Arithmetic
Immediate Arithmetic
Load
Store
Branch
```
然后生成大部分的 `Datapath Control Signals`

## ALU Decoder
---
ALU 译码器 进一步结合：
- ALUOp
- funct3
- funct7
决定：
```
ADD
SUB
AND
OR
XOR
SLT
...
```
最终生成：`ALUControl`

## ALUOp
---
ALU Operation Category ALU操作类别
**并不是 ALUControl**
我们可以设计
```
ALUOp = 00 -> 强制ADD
ALUOp = 01 -> Branch Compare
ALUOp = 10 -> 根据 funct 字段继续判断
```
然后
```
ALUOp + funct3 + funct7
↓ ALU Decoder
ALUControl
```

**从Main Decoder开始 都是属于微架构环节**

## Control Logic 大部分属于 组合逻辑
---
对于 简单 Single-Cycle RISC-V CPU:
大部分 Instruction Decode / Control Logic 可以看成 Combination Logic

也就是：
```
Instruction Bits
       ↓
Combination Decode Logic
       ↓
Control Signals
```
Instruction 改变后 经过传播延迟：
`Control Signals` 随之改变

## 但是 CPU仍是时序电路
---
整个结构是：
```
Current State
      │
      ▼
Combinational Datapath + Control
      │
      ▼
Next State
      │
      ▼
Clock Edge
      │
      ▼
New State
```
**CPU = State状态 + Combinational Logic 组合逻辑 + Clocked State Update 时钟状态更新**
本质上 是同步数字系统


