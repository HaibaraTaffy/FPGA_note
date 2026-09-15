## 运算器
---
![[Pasted image 20260906172225.png]]
核心部件 ALU 即算术逻辑单元 主要采用组合逻辑的设计方式
其余是一些寄存器
MQ 乘商寄存器 主要负责数据的乘法与除法运算并保存结果
ACC 累加器
X 通用寄存器
PSW 程序状态字寄存器 存放状态信息和控制信息 可以提供给控制器进行下一步指令的判断
## 控制器
---
![[Pasted image 20260906205110.png]]
主要结构 CU 控制单元 由控制状态和组合控制逻辑等共同完成指令控制
IR **指令寄存器** 存放当前执行的指令
PC **程序计数器** 存放取指所需的指令地址 顺序执行时会按指令长度更新 遇到分支或跳转时则更新为目标地址


## 符号约定 微操作记法
---
M : 代表主存 或 存储体中的某个存储单元
ACC MQ X MAR MDR : 表示相应的寄存器
(寄存器) : 访问该寄存器内的数据
M(MAR) : 访问存储体位于MAR那个存储单元里的数据

指令 :  由操作码 + 地址码组成
操作码 : 代表指令操作的序列
地址码 : 代表需要操作的数据在哪个位置
OP(IR) : 取指令寄存器中的操作码
AD(IR) : 取指令寄存器中的地址码




## 取数指令
---
`(PC) -> MAR` 把PC中存放的地址(下一条指令的地址) 直接复制一份 存到 MAR这个寄存器中
`M(MAR) -> MDR` 把MAR里存放的那个地址所对应的主存单元里的数据 (**这一次取得是指令**) 取出来 放到MDR中去
`(MDR) -> IR` 把MDR中的数据(指令) 放到IR(指令寄存器)中
**取指令结束**

`OP(IR) -> CU` 把IR中的操作码 取出 进入CU的译码逻辑中 进行解析
**分析指令结束** 在当下情景 是一个取数指令

`AD(IR) -> MAR` 把IR中的地址码 取出 放到MAR中
`M(MAR) -> MDR` 把MAR里存放的那个地址所对应的主存单元里的数据 (**这一次取的是数据**) 取出来 放到MDR中去
`(MDR) -> ACC` 把MDR中的数据 放入ACC寄存器中
**执行指令结束**

MDR中的数据 放入IR还是ACC 取决于MDR中存放的 是指令还是数据

**CPU区分指令和数据的依据** : 指令周期的不同阶段 
如何知道是不同阶段?设置了相应的寄存器 通过查询寄存器的状态 可以知道是什么阶段 然后根据这个阶段来决定最终的数据流向

---
**以上是传统教材中的 "累加器型CPU" 资料来源 王道考研计组课程**

**以下是GPT的RISC-V 教程**

## CPU Execution Process
---
CPU 执行一条指令 可以从功能上划分为 **五个阶段** :
- **IF — Instruction Fetch — 指令取指**
- **ID — Instruction Decode — 指令译码**
- **EX — Execute — 执行**
- **MEM — Memory Access — 存储器访问**
- **WB — Write Back — 写回**

仅是功能阶段 并不代表固定需要 五个 Clock Cycle

根据实现的不同 可以划分为 不同的时间组织与对应微架构 :

|微架构|五个功能阶段如何执行|
|---|---|
|Single-Cycle CPU|一条指令在一个 Clock Cycle 内完成全部阶段|
|Multi-Cycle CPU|一条指令分多个 Clock Cycle 完成|
|Pipeline CPU|多条指令的不同阶段重叠执行|
## IF 指令取指
---
解决 : **当前应该从哪里取得哪条指令**
主要数据路径：
```
PC
 ↓ Instruction Address
Instruction Memory
 ↓
Instruction
```
同时通常还会计算顺序执行地址：
```
PC + 4
```
例如：
`lw x5, 8(x6)`
假设当前：
`PC = 0x1000`
那么 IF 阶段完成：
```
Instruction Memory[0x1000]
取出→ lw x5, 8(x6)

Sequential Next PC
= 0x1000 + 4
= 0x1004
```
此时的 `PC + 4` 只是一个 Next PC 候选值 遇到 Branch 或 Jump 时 最终 Next PC 可能选择其他地址

## ID 指令译码
---
ID阶段 解决:
1. 当前是什么指令
2. 需要读取哪些 Register
3. Datapath 应该如何配置

对：
```
lw x5, 8(x6)
```
译码后得到：
```
rs1 = x6
rd  = x5
Immediate = 8
Instruction Class = Load
```
数据部分：
```
Register File:
RF[x6] → Read Data 1

ImmGen:
Immediate Field → 32-bit Immediate
```
控制部分产生：
```
RegWrite  = 1
ALUSrc    = 1
ALUControl = ADD
MemWrite  = 0
ResultSrc = Memory
```

## EX 执行
---
**使用ALU Comparator Adder 等组合逻辑完成指令要求的运算**
不同指令 在EX阶段 的任务不同:

|指令|EX 阶段的主要工作|
|---|---|
|`add`|计算 `RF[rs1] + RF[rs2]`|
|`addi`|计算 `RF[rs1] + Immediate`|
|`lw`|计算 Effective Address|
|`sw`|计算 Effective Address|
|`beq`|比较两个 Register Value，并计算 Branch Target|
## MEM 存储器访问
---
**访问 Data Memory**
例如
### `lw`
```
ALU Result
→ Data Memory Address
→ Memory Read Data
```
例如：
```
Memory[0x1008] = 123
```
则 MEM 阶段得到：
```
Memory Read Data = 123
```
### `sw`
```
sw x5, 8(x6)
```
数据路径为：
```
ALU Result → Data Memory Address
RF[x5]     → Data Memory Write Data
MemWrite   → Data Memory Write Enable
```

**并不是每条指令 都会产生有效的 Memory Access**

## WB 写回
---
解决 : **是否更新 Register File?写入哪个 Register?写入什么数据?**
例如：
```
lw x5, 8(x6)
```
对应：
```
Write Address = rd = x5
Write Data    = Memory Read Data
RegWrite      = 1
```
于是：
```assembly
RF[x5] ← Memory Read Data
```

## 一条 lw 的完整过程
---
`lw x5, 8(x6)`
假设：
```
PC = 0x1000
RF[x6] = 0x2000
Memory[0x2008] = 100
```

|阶段|数据流与操作|
|---|---|
|IF|用 `PC=0x1000` 读取指令，同时计算 `PC+4`|
|ID|得到 `rs1=x6`、`rd=x5`、`Immediate=8`，读取 `RF[x6]`|
|EX|ALU 计算 `0x2000+8=0x2008`|
|MEM|读取 `Memory[0x2008]`，得到 `100`|
|WB|将 `100` 写入 `RF[x5]`|

最终状态变化：
```
PC    ← 0x1004
RF[x5] ← 100
```
没有改变的内容包括：
```
RF[x6]
Memory[0x2008]
```
因为 `lw` 只读 Data Memory 不写 Data Memory

一条指令的完整阐述

**Instruction Encoding 经过 Decode 产生 Control Signals Control Signals 配置 Datapath 使数据经过指定硬件形成Next State**

## 五类指令 阶段对照
---

|阶段|`add`|`addi`|`lw`|`sw`|`beq`|
|---|---|---|---|---|---|
|IF|取指|取指|取指|取指|取指|
|ID|读两个 RF 数据|读 RF、生成立即数|读基址、生成偏移量|读基址和待写数据|读两个 RF 数据|
|EX|加法|加法|计算地址|计算地址|比较并计算目标地址|
|MEM|无有效访问|无有效访问|读取 Memory|写入 Memory|无有效访问|
|WB|写 ALU Result|写 ALU Result|写 Memory Read Data|不写 RF|不写 RF|

