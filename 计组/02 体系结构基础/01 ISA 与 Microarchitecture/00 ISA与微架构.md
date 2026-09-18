| 缩写      | 英文全称                         | 中文     |
| ------- | ---------------------------- | ------ |
| **ISA** | Instruction Set Architecture | 指令集架构  |
| **CPU** | Central Processing Unit      | 中央处理器  |
| **ALU** | Arithmetic Logic Unit        | 算术逻辑单元 |
| **PC**  | Program Counter              | 程序计数器  |
| **RF**  | Register File                | 寄存器堆   |
| **RTL** | Register Transfer Level      | 寄存器传输级 |
|         | Microarchitecture            | 微架构    |

## 层次
---
```
高级程序
C / C++ / Python ...
        ↓
      编译器
        ↓
ISA（指令集架构）
        ↓
Microarchitecture（微架构）
        ↓
RTL
        ↓
数字电路
        ↓
晶体管 / FPGA物理资源
```

## Instruction
---
即 指令 CPU能够识别并执行的一个基本操作
```
ADD 加法
SUB 减法
LOAD 从存储器读取数据
STORE 向存储器写入数据
```

指令实际上也是一串bit

- Operation Code : 操作码 表示 这条指令要执行什么操作
- Operand : 操作数 表示 这个操作要处理谁

**一条机器指令 本质上就是 Opcode + Operand**

## ISA
---
即 Instruction Set Architecture 指令集架构
是 **软件和处理器硬件之间 约定的一整套规则**
我们以 开源的 RISC-V 为例 来进行学习!!!

```
例如 ISA 会规定：
- 有哪些指令；
- 指令如何编码；
- 有哪些程序可见寄存器；
- 数据类型和操作方式；
- Load/Store 如何访问存储器；
- Branch 如何改变程序执行流程；
- 异常等体系结构行为。
```

可以说 ISA是软件与处理器硬件之间的重要抽象接口
于是 硬件设计者只需要保证 从软件角度观察 指令执行后的结果 符合预期即可
于是 **ISA 相同 不代表CPU内部硬件结构相同**

## Microarchitecture
---
微架构 体现了 **处理器内部具体怎样组织硬件 从而实现ISA**

例如：
```
ISA规定：
ADD R3,R1,R2
```
微架构则需要考虑：
```
R1、R2存在哪里？
        ↓
怎么读取？
        ↓
送到哪个ALU？
        ↓
经过几个Pipeline Stage？
        ↓
什么时候产生结果？
        ↓
怎么写回R3？
```

## Register 和 Register File
---
- Register 寄存器
- Register File 寄存器堆
Register File 就是 很多Register 组织在一起
即一组有组织的寄存器集合

## CPU抽象执行流程
---
假设：
```
ADD R3,R1,R2
```
第一步，CPU取得 Instruction：
```
ADD R3,R1,R2
```
然后识别：
```
Opcode = ADD
```
并知道 Operand：
```
Source 1 = R1
Source 2 = R2
Destination = R3
```
于是：
```
          RF
     ┌───────────┐
     │ R1 = 5    │────┐
     │ R2 = 7    │──┐ │
     │ R3        │  │ │
     └───────────┘  │ │
                    ▼ ▼
                   ┌───┐
                   │ALU│
                   └─┬─┘
                     │
                    12
                     │
                     ▼
                  写入 R3
```
最终 : R3=12

## Control
---
ALU需要控制逻辑 指导 概念上有
```
Instruction
     ↓
  Decode
     ↓
Control Logic
     ↓
┌────┴────────────┐
│                 │
▼                 ▼
RF Read        ALU Operation
                  │
                  ▼
                 ADD
```

- Decode : 译码 根据指令编码判断 这是什么指令?需要哪些硬件行为?

于是微架构中 有一个经典的划分 `Datapath + Control`
即
```
CPU
│
├─ Datapath
│    └─ 数据存储、移动、计算
│
└─ Control
     └─ 决定这些硬件什么时候做什么
```

## CPU 与 AI Accelerator
---
CPU 执行普通的 Instruction
而AI Accelerator 则可能提供专门的
- Matrix Operation
- Vector Operation
- Tensor Operation

**硬件擅长什么计算 往往会反映到软件能够调用什么操作**

## Instruction Set
---
即 指令集合 
与ISA的关系
```
ISA
│
├─ Instruction Set
├─ Register
├─ Instruction Encoding
├─ Data Type
├─ Memory Access Rules
├─ Addressing相关规则
├─ Exception行为
└─ ...
```

## RISC CISC
---
- RISC 即Reduced Instruction Set Computer 精简指令集计算机
- CISC 即Complex Instruction Set Computer 复杂指令集计算机

```
粗略理解
RISC
→ 倾向较规则、较简单的指令
→ 强调组合简单操作完成复杂任务

CISC
→ 指令功能和编码形式可能更复杂
→ 单条指令可能完成更多工作
```

