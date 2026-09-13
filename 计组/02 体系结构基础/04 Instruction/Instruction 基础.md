## Instruction
---
本质上是 告诉CPU **对哪些数据 执行什么操作 结果放在哪里**
例如 RISC-V中的
```
add x5, x6, x7
```
意思是 `读取x6 读取x7 执行加法 结果写入x5`

## 汇编
---
即Assembly Language 汇编语言 例如上面的那一条

而RISC-V的这条`add` 最终对应一个 32bit的 **Instruction Encoding 即指令编码**
 Assembly Instruction 通过 **Assembler 汇编器** 转换成 **Machine Instruction Encoding 机器指令编码**

## Instruction Encoding
---
指令编码 即ISA规定的 如何用一串bit表示一条Instruction
例如 : RV32I

## RV32I
---
先解释名字：
- `RV`：RISC-V
- `32`：基础整数寄存器宽度为 32 bit
- `I`：Base Integer Instruction Set —— 基础整数指令集
RV32I 的基础指令通常使用：`32bit 即4Byte` Instruction Encoding

## Instruction Format
---
指令格式 由ISA进行规定的bit字端划分方式
哪些bit表示哪些 ...
RV32I 有几种主要格式：
```
R-type
I-type
S-type
B-type
U-type
J-type
```

由于不同指令类型对bit字段的需求不同 于是
**同一个ISA内部会设计多种 Instruction Format**

## Field
---
字段 即 Instruction Encoding 中承担特定意义的一组bit
例如 RISC-V 的一条 Instruction 中可能有：
```
opcode
rd
rs1
rs2
funct3
funct7
immediate
```
这些都是字段
**字段是指令编码中的一部分**

## Opcode
---
Operation Code 操作码 告诉CPU
**这条 Instruction 属于什么基本操作类别**
例如 `add sub` 都属于 `寄存器-寄存器整数运算`这一类
但仅靠一个简单的大类 Opcode 有时还不足以区分到底是：
```
ADD
SUB
AND
OR
XOR
```
因此 RISC-V 还会使用：
```
funct3
funct7
```
进一步区分具体操作
于是 Opcode不一定独自完整决定一条指令是什么
**Instruction 的具体语义由相关编码字段共同确定**

## Operand
---
操作数 即 Instruction 要处理的数据 一般是寄存器内的值

例如：
```
add x5, x6, x7
```
语义为 : `RF[x5] <- RF[x6] + RF[x7]`
- `x6` 是 **Source Register 1 — 第一源寄存器**
- `x7` 是 **Source Register 2 — 第二源寄存器**
- `x5` 是 **Destination Register — 目的寄存器**

`RF[x6] RF[x7]` 都是**源操作数** x5是目的寄存器

## Register Name Register Index 和 Register Value
---
- **Register Name** : 寄存器名称
  RISC-V 把整数寄存器命名为：
```
x0
x1
x2
...
x31
```
所以 `x5` `x6` `x7` 都是寄存器名称

- **Register Index** : 寄存器编号
	RISC-V 一共有 32 个整数寄存器
	所以：
	- `x5` 的 Register Index 是 5
	- `x6` 的 Register Index 是 6
	- `x7` 的 Register Index 是 7
	因为一共有 32 个寄存器：
	所以只需要 **5 bit** 就可以指定其中任意一个寄存器(0-31)
	例如：如果指令中 `rs1 = 00110`，表示选择：`x6`

- **Register Value** : 寄存器值
  一般用 `RF[x_index(十进制)]` 来表示 是真正参与计算的数据

## Register Specifier
---
寄存器指定字段 **用来指定 要访问哪个 Register**
还是经典的指令为例
```
add x5, x6, x7
```

- **rs1** 即 Source Register 1 : 第一源寄存器 在这里是 **x6**
- **rs2** 即 Source Register 2 : 第二源寄存器 在这里是 **x7**
- **rd**  即 Destination Register : 目的寄存器 在这里是 **x5**
这些都是 **寄存器编号** 而不是 数据
可以联系 **2R1W**

## x0 RISC-V的零寄存器
---
在 RISC-V 中 `x0` 是一个特殊寄存器
读取它 永远得到0 
即使执行：
```
add x0, x5, x6
```
计算结果也不会真正改变 x0
可以方便实现很多常见操作 比如复制
`add x5, x6, x0` 可以实现把 `x6` 的值复制到 `x5`

## Immediate
---
立即数 是直接编码在 Instruction 中的常数值
例如：**注意 这里用的是 addi 而非 add**
```
addi x5, x6, 10
```
它的语义是：`RF[x5] <- RF[x6] + 10`
这里的 10 就是 Immediate 数据直接来自 Instruction

于是又可以区分出 :
- Register Operand : 数据来自 Register
- Immediate Operand : 数据直接来自 Instruction

## ALU前的MUX
---
解决
```
RF Read Data 2 ──────┐
                     │
Immediate ───────────┤
                     ▼
                    MUX
                     │
                     ▼
               ALU Operand B
```
问题

## Immediate 扩展
---
在 RV32I 中 整数 Datapath 通常处理 `32bit`
由于 Instruction 的长度有限 `32bit` 还需要容纳其他字段 所以Immediate 通常不可能占据完整的32bit
CPU内部 就需要将其变成适合 `32bit` Datapath 使用的数据

## Sign Extension
---
即符号扩展 在Immediate 表示有符号数时 是经常需要的
可以参考之前的
[mynote/计组/01 数据表示与可靠性/数值]

## Immediate Generator
---
立即数生成器 简称 **ImmGen**
它负责 从Instruction 中提取 Immediate
其功能可能包括：
```
Instruction
    ↓
提取 Immediate Bits
    ↓
重新组合
    ↓
Sign Extension
    ↓
32-bit Immediate
```
**RISC-V的不同 Instruction Format 中 Immediate的bit位置并不完全相同**
所以 ImmGen 不一定只是简单的符号扩展

