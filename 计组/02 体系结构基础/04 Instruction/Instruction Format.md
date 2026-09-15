以RV32I 的 六种主要格式为例
```
R-type
I-type
S-type
B-type
U-type
J-type
```

## Instruction Format
---
指令格式 即表示一条 32 bit Instruction 内部各个 bit Field 如何划分 以及每个字段表示什么

虽然每个 Instruction 的长度相同 均为 32bit
但是 **不同指令需要的信息不同**
所以不能使用完全相同的字段分配

RV32I定义了六种主要格式：

```
R-type
I-type
S-type
B-type
U-type
J-type
```

## 六种 Type
---

| Type   | 英文含义                 | 主要用途              |
| ------ | -------------------- | ----------------- |
| R-type | Register Type        | Register之间的运算     |
| I-type | Immediate Type       | 立即数运算、Load、`jalr` |
| S-type | Store Type           | Store             |
| B-type | Branch Type          | 条件Branch          |
| U-type | Upper Immediate Type | 高位立即数             |
| J-type | Jump Type            | `jal`跳转           |
|        |                      |                   |
## 共同字段
---
RV32I 尽量让常用字段 固定在相同位置

| 字段       | Bit位置     | 含义                         |
| -------- | --------- | -------------------------- |
| `opcode` | `[6:0]`   | 指令大类                       |
| `rd`     | `[11:7]`  | Destination Register Index |
| `funct3` | `[14:12]` | 进一步区分操作                    |
| `rs1`    | `[19:15]` | Source Register 1 Index    |
| `rs2`    | `[24:20]` | Source Register 2 Index    |
| `funct7` | `[31:25]` | 进一步区分操作                    |

## R-type
---
主要用于 两个 Register 之间的运算
例如
```
add x5, x6, x7
sub x5, x6, x7
and x5, x6, x7
or  x5, x6, x7
xor x5, x6, x7
```

字段布局 :

|Bit|`[31:25]`|`[24:20]`|`[19:15]`|`[14:12]`|`[11:7]`|`[6:0]`|
|---|---|---|---|---|---|---|
|字段|`funct7`|`rs2`|`rs1`|`funct3`|`rd`|`opcode`|
**R-type 没有 Immediate 因此不需要 ImmGen**

## I-type
---
I-type包含一个 12-bit Immediate **写成16进制 就是指令的最高三位**
例如
```
addi x5, x6, 10
lw   x5, 8(x6)
jalr x1, 0(x5)
```

字段布局 :

|Bit|`[31:20]`|`[19:15]`|`[14:12]`|`[11:7]`|`[6:0]`|
|---|---|---|---|---|---|
|字段|`imm[11:0]`|`rs1`|`funct3`|`rd`|`opcode`|
于是 立即数扩展可以写成
```Verilog
{{20{instruction[31]}},instruction[31:20]}
```

## S-type
---
专门用于 Store
例如`sw x5, 8(x6)`

```
rs1：提供Base Address
rs2：提供待写入Memory的数据
Immediate：提供Address Offset
```

字段布局:

|Bit|`[31:25]`|`[24:20]`|`[19:15]`|`[14:12]`|`[11:7]`|`[6:0]`|
|---|---|---|---|---|---|---|
|字段|`imm[11:5]`|`rs2`|`rs1`|`funct3`|`imm[4:0]`|`opcode`|

于是ImmGen 需要重新拼接Immediate 再进行符号位扩展

```Verilog
immediate = {
                {20{instruction[31]}},
                instruction[31:25],
                instruction[11:7]
            };
```

## B-type
---
用于条件分支 例如
```
beq x5, x6, target
```

字段布局 : 

| Bit | `[31]`    | `[30:25]`   | `[24:20]` | `[19:15]` | `[14:12]` | `[11:8]`   | `[7]`     | `[6:0]`  |
| --- | --------- | ----------- | --------- | --------- | --------- | ---------- | --------- | -------- |
| 字段  | `imm[12]` | `imm[10:5]` | `rs2`     | `rs1`     | `funct3`  | `imm[4:1]` | `imm[11]` | `opcode` |

B-type Immediate需要重组为：
```
imm[12]
imm[11]
imm[10:5]
imm[4:1]
imm[0]
```
对应：
```Verilog
{
    instruction[31],
    instruction[7],
    instruction[30:25],
    instruction[11:8],
    1'b0
}
```
最低位：
```
imm[0] = 0
```
因此 Branch Offset一定是2的整数倍。

B-type 使用 **PC-Relative Addressing PC相对寻址**
目标地址为 `BranchTarget = CurrentPC + Immediate`

执行逻辑：
```
如果RF[x5] == RF[x6]
    Next PC = Current PC + Branch Immediate
否则
    Next PC = Current PC + 4
```

## U-type
---
用于较大的高位立即数
主要指令
```
lui
auipc
```

字段布局 :

| Bit | `[31:12]`    | `[11:7]` | `[6:0]`  |
| --- | ------------ | -------- | -------- |
| 字段  | `imm[31:12]` | `rd`     | `opcode` |
生成的32-bit Immediate为：
```Verilog
{
    instruction[31:12],
    12'b0
}
```
也就是把 Instruction中的20-bit Immediate放到结果高20 bit 低12 bit补0

## J-type
---
主要用于
```
jal x1, target
```

字段布局

|Bit|`[31]`|`[30:21]`|`[20]`|`[19:12]`|`[11:7]`|`[6:0]`|
|---|---|---|---|---|---|---|
|字段|`imm[20]`|`imm[10:1]`|`imm[11]`|`imm[19:12]`|`rd`|`opcode`|

重新排列：
```Verilog
{
    instruction[31],
    instruction[19:12],
    instruction[20],
    instruction[30:21],
    1'b0
}
```

J-type同样使用 PC-Relative Addressing：
`JumpTarget=CurrentPC+Immediate`
同时保存返回地址：
`RF[rd]←CurrentPC+4`
因此 `jal`既会改变 PC，也会写 Register File。

## 核心结论
---
**Instruction Type 决定 32bit 指令内部的字段布局
Instruction Type 不完全等于指令功能类别
不同Type的Immediate 位置不同 因此需要ImmGen重组**

