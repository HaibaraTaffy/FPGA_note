## 本课目标
---
本课对完整的 Single-Cycle RV32I Mini Core 进行 CPU-Level Test——CPU 级测试。
与此前的模块级测试不同，本课不是分别验证 ALU、Register File 或 Decoder，而是让 CPU 连续执行一段完整程序，观察各模块组合起来后能否正确完成：
- Instruction Fetch
- Instruction Decode
- Register File Read
- Execute
- Memory Access
- Write Back
- Next-PC Update
测试程序覆盖当前 CPU 支持的全部指令：
- `addi`
- `add`
- `sub`
- `and`
- `or`
- `xor`
- `sw`
- `lw`
- `beq` 条件不成立
- `beq` 条件成立

## 测试程序
---
```
addi x1,  x0, 5
addi x2,  x0, 3

add  x3,  x1, x2
sub  x4,  x1, x2
and  x5,  x1, x2
or   x6,  x1, x2
xor  x7,  x1, x2

sw   x3, 0(x0)
lw   x8, 0(x0)

addi x10, x0, 1

beq  x1, x2, +8
addi x9, x0, 9

beq  x3, x8, +8
addi x10, x0, 10

addi x11, x0, 11
```

## 地址与机器码
---

|Byte Address|Assembly|Machine Code|
|---|---|---|
|`0x00000000`|`addi x1, x0, 5`|`00500093`|
|`0x00000004`|`addi x2, x0, 3`|`00300113`|
|`0x00000008`|`add x3, x1, x2`|`002081B3`|
|`0x0000000C`|`sub x4, x1, x2`|`40208233`|
|`0x00000010`|`and x5, x1, x2`|`0020F2B3`|
|`0x00000014`|`or x6, x1, x2`|`0020E333`|
|`0x00000018`|`xor x7, x1, x2`|`0020C3B3`|
|`0x0000001C`|`sw x3, 0(x0)`|`00302023`|
|`0x00000020`|`lw x8, 0(x0)`|`00002403`|
|`0x00000024`|`addi x10, x0, 1`|`00100513`|
|`0x00000028`|`beq x1, x2, +8`|`00208463`|
|`0x0000002C`|`addi x9, x0, 9`|`00900493`|
|`0x00000030`|`beq x3, x8, +8`|`00818463`|
|`0x00000034`|`addi x10, x0, 10`|`00A00513`|
|`0x00000038`|`addi x11, x0, 11`|`00B00593`|

Instruction Memory 使用 Word Index 访问 因此 Hex 文件中的第一个机器码对应地址 `0x00` 之后每行地址增加 4 Byte

## 算术与逻辑指令的数据路径
---
### R-type 指令
`add`、`sub`、`and`、`or` 和 `xor` 使用两项 Register File 数据：
```
Instruction
→ rs1、rs2、rd
→ Register File
→ RF[rs1]、RF[rs2]
→ ALU
→ ALU Result
→ Write-Back MUX
→ RF[rd]
```
其中：
- `rs1`、`rs2` 和 `rd` 是 Register Index。
- `RF[rs1]` 和 `RF[rs2]` 才是实际的 Operand Value。
- `alu_control` 决定 ALU 执行哪种运算。
### addi
`addi` 的 Operand A 来自 `RF[rs1]`，Operand B 来自 Immediate：
```
RF[rs1] + Immediate
→ ALU Result
→ Write-Back MUX
→ RF[rd]
```
此时：
```
alu_src = 1
result_src = 0
reg_write_enable = 1
```

## Data Memory 访问
---
### sw
`sw` 使用 ALU 计算 Memory Address：
```Memory Address = RF[rs1] + Immediate```
真正写入 Data Memory 的数据来自：
```Memory Write Data = RF[rs2]```
本次测试执行：
```sw x3, 0(x0)```
因此：
```
Memory Address = RF[x0] + 0 = 0
Memory Write Data = RF[x3] = 8
```
在有效时钟上升沿：
```Data Memory[0] ← 8```

### lw
本次测试随后执行：
```lw x8, 0(x0)```
数据路径为：
```
RF[x0] + 0
→ ALU Result
→ Memory Address
→ Data Memory
→ memory_read_data
→ Write-Back MUX
→ RF[x8]
```
当前 Data Memory 采用组合读取，但 Register File 的写回仍发生在时钟上升沿。
执行完成后：
```RF[x8] = 8```

## Branch 测试
---
### beq 条件不成立
第一条 Branch 指令为：
```beq x1, x2, +8```
此时：
```
RF[x1] = 5
RF[x2] = 3
```
ALU 执行减法：
```5 - 3 = 2```
因此：
```
zero = 0
branch = 1
pc_src = branch & zero = 0
```
PC 选择 `pc_plus4`，继续执行地址 `0x2C` 的：
```addi x9, x0, 9```

### beq 条件成立
第二条 Branch 指令为：
```beq x3, x8, +8```
此时：
```
RF[x3] = 8
RF[x8] = 8
```
ALU 执行减法：
```8 - 8 = 0```
因此：
```
zero = 1
branch = 1
pc_src = 1
```
Branch Target 为：
```
0x30 + 8 = 0x38
```
所以 PC 从 `0x30` 直接更新为 `0x38`，跳过地址 `0x34` 的：
```
addi x10, x0, 10
```

## 验证被跳过的指令没有执行
---
在 Branch 之前，程序先执行：
```addi x10, x0, 1```
因此：
```RF[x10] = 1```
如果地址 `0x34` 的指令被错误执行，`RF[x10]` 就会变成 10。
实际波形中：
```
PC：0x30 → 0x38
RF[x10]：保持为 1
```
说明地址 `0x34` 的指令没有执行。

## State、Data 与 Control
---
### State
当前 CPU 中会跨时钟周期保存的信息包括：
- PC Register 中的 PC。
- Register File 中的通用寄存器值。
- Data Memory 中的数据。
- Instruction Memory 中预先加载的程序。

### Data
指令执行过程中流动的数据包括：
- Instruction
- RF[rs1]
- RF[rs2]
- Immediate
- ALU Operand
- ALU Result
- Memory Read Data
- Write-Back Data
- Next PC

### Control
控制当前指令实际使用哪条数据路径的信号包括：
- `reg_write_enable`
- `alu_src`
- `mem_write_enable`
- `result_src`
- `branch`
- `alu_control`
- `pc_src`

Datapath 提供可使用的数据路径，Control 决定当前指令选择哪条路径。

## 时钟边沿与状态更新
---
一个单周期的执行过程可以表示为：
```
Current State
→ Combinational Datapath + Control
→ Next State
→ Clock Edge
→ New State
```
在一个周期内，Instruction、Decoder、Register File 读取端、Immediate Generator、MUX、ALU、Memory 读取端和 Next-PC Logic 都属于组合计算过程。

在有效时钟上升沿：
- PC 捕获 `next_pc`。
- `reg_write_enable=1` 时，Register File 写入 `write_back_data`。
- `mem_write_enable=1` 时，Data Memory 写入 `read_data2`。

## 无效组合信号
---
组合逻辑会持续计算输出，所以某些没有被当前指令使用的信号也可能出现具体数值或 `X`。

例如在 `addi` 中：
- `instruction[24:20]` 仍然会被截取为 `rs2`。
- Register File 仍会根据这个 `rs2` 产生 `read_data2`。
- 但 `alu_src=1`，ALU Operand B 选择的是 Immediate。
- 因此 `read_data2` 不参与当前有效数据路径。

判断一个异常值是否影响 CPU，应当检查：
1. 它是否被有效 MUX 选中。
2. 它是否参与有效控制决策。
3. 它是否在时钟上升沿造成状态更新。

## zero 的红色短线
---
ALU 中：
```assign zero = (result == 32'b0);```
因此：
```
ALU Operand / ALU Control
→ alu_result
→ zero
```
在指令切换时，不同组合逻辑需要经过若干 Delta Cycle 才能全部稳定。

例如从 `beq` 切换到 `addi` 时：
1. 新指令的 `rs2` 可能先指向一个未初始化寄存器。
2. `read_data2` 暂时为 `X`。
3. `alu_src` 可能还没有从旧值更新到新值。
4. `alu_operand_b` 暂时选择到 `X`。
5. `alu_result` 短暂变为 `X`。
6. `zero` 也短暂变为 `X`。
7. 所有组合逻辑稳定后，信号恢复正确。

只要这些信号在下一个时钟上升沿之前稳定，就不会造成错误的状态更新。
本次波形中的红色短线属于组合逻辑的 Delta Cycle 瞬态，不是 ALU 功能错误，也不是 ModelSim Bug。

## 固定波形信号组
---
项目中已经建立：
`sim/modelsim/cpu_wave.do`
固定观察：
### Clock 与取指
- `clk`
- `rst`
- `pc`
- `instruction`
- `next_pc`
### Instruction Fields
- `opcode`
- `rs1`
- `rs2`
- `rd`
- `immediate`
### Register File 与 ALU
- `read_data1`
- `read_data2`
- `alu_operand_b`
- `alu_control`
- `alu_result`
- `zero`
### Memory 与 Write Back
- `memory_read_data`
- `write_back_data`
- `reg_write_enable`
- `mem_write_enable`
### MUX 与 Branch
- `alu_src`
- `result_src`
- `branch`
- `pc_src`
- `branch_target`

后续执行当前单周期 CPU 的不同程序时，不需要再逐条指令添加波形信号。

## 本课结论
---
本课完成了从模块级仿真到 CPU 级完整程序仿真的过渡。
当前 CPU 已经能够连续完成：
```
取指
→ 译码
→ 读取操作数
→ ALU 运算
→ Data Memory 访问
→ Register File 写回
→ 条件分支
→ PC 更新
```
这说明当前模块已经组成一颗能够执行基础程序的 Single-Cycle RV32I Mini Core

___

## V2扩展
---
在原先 指令的基础上 扩展新指令
```
SLL
SRL
SRA
SLT

BNE
BLT
BGE

JAL
JALR
```

现有测试指令

|   PC | Instruction       | 解释                                                 | 结果                                  |
| ---: | ----------------- | -------------------------------------------------- | ----------------------------------- |
| `00` | `addi x1 x0 5`    | `RF[x0] + 5 存入 x1`                                 | `RF[x1] = 5`                        |
| `04` | `addi x2 x0 3`    | `RF[x0] + 3 存入 x2`                                 | `RF[x2] = 3`                        |
| `08` | `add x3 x1 x2`    | `RF[x1] + RF[x2] 存入 x3`                            | `RF[x3] = 8`                        |
| `0C` | `sub x4 x1 x2`    | `RF[x1] - RF[x2] 存入 x4`                            | `RF[x4] = 2`                        |
| `10` | `and x5 x1 x2`    | `RF[x1] & RF[x2] 存入 x5`                            | `RF[x5] = 1`                        |
| `14` | `or x6 x1 x2`     | `RF[x1] \| RF[x2] 存入 x6`                           | `RF[x6] = 7`                        |
| `18` | `xor x7 x1 x2`    | `RF[x1] ~\| RF[x2] 存入 x7`                          | `RF[x7} = 6`                        |
| `1C` | `sw x3 0(x0)`     | `将RF[x3] 存入 0地址`                                   | `Memory[0] = 8`                     |
| `20` | `lw x8 0(x0)`     | `将 Memory[0] 载入 x8`                                | `RF[x8] = 8`                        |
| `24` | `addi x10 x0 1`   | `RF[x0] + 1 存入 x10`                                | `RF[x10] = 1`                       |
| `28` | `beq x1 x2 +8`    | `比较 RF[x1] 和 RF[x2] 相等就跳转`                         | `不相等 pc_next = pc + 4`              |
| `2C` | `addi x9 x0 9`    | `RF[x0] + 9 存入 x9`                                 | `RF[x9] = 9`                        |
| `30` | `beq x3 x8 +8`    | `比较 RF[x3] 和 RF[x8] 相等就跳转`                         | `相等 next_pc = pc + Immediate =38`   |
| `34` | `addi x10 x0 10`  | `RF[x0] + 10 存入 x10`                               | `但其实被跳过`                            |
| `38` | `addi x11 x0 11`  | `RF[x0] + 11 存入 x11`                               | `RF[x11] = 0xB`                     |
| `3C` | `addi x12 x0 -8`  | `RF[x0] - 8 存入 x12`                                | `RF[12] = -8 = 0xffff_fff8`         |
| `40` | `addi x13 x0 35`  | `RF[x0] + 35 存入 x13`                               | `RF[13] = 0x23 注意进制转换`              |
| `44` | `sll x14 x1 x13`  | `RF[x1] 左移 RF[x13][4:0] 存入 x14`                    | `相当于左移三位 RF[x14] = 0x28`            |
| `48` | `srl x15 x12 x13` | `RF[x12] 逻辑右移 RF[x13][4:0] 存入x15`                  | `相当于补零右移三位 RF[x15] = 0x1fff_ffff`   |
| `4C` | `sra x16 x12 x13` | `RF[x12] 算术右移 RF[x13][4:0] 存入x16`                  | `相当于补符号位右移三位 RF[x16] = 0xffff_ffff` |
| `50` | `slt x17 x12 x1`  | `比较 RF[x12] 与 RF[x1] 小于则置 x17 的位`                  | `小于 RF[x17] = 1`                    |
| `54` | `slt x18 x1 x12`  | `比较 RF[x1] 与 RF[x12] 小于则置 x18 的位`                  | `不小于 RF[x18] = 0`                   |
| `58` | `bne x1 x2 +8`    | `比较 RF[x1] 和 RF[x2] 不相等就跳转`                        | `不相等 pc_next = pc + Immediate`      |
| `5C` | `addi x19 x0 19`  | `RF[x0] + 19 存入 RF[x19]`                           | `被跳过 RF[x19] = 0`                   |
| `60` | `bne x1 x1 +8`    | `比较 RF[x1] 和 RF[x2] 不相等就跳转`                        | `相等 pc_next = pc + 4`               |
| `64` | `addi x19 x0 20`  | `RF[x0] + 20 存入 RF[x19]`                           | `RF[x19] = 0x14`                    |
| `68` | `blt x12 x1 +8`   | `比较 RF[x12] 和 RF[x1] 小于就跳转`                        | `小于 pc_next = pc + Immediate`       |
| `6C` | `addi x20 x0 20`  | `RF[x0] + 20 存入 RF[x20]`                           | `被跳过 RF[x20] = 0`                   |
| `70` | `bge x1 x12 +8`   | `比较 RF[x1] 和 RF[x12] 大于等于就跳转`                      | `大于 pc_next = pc + Immediate`       |
| `74` | `addi x20 x0 21`  | `RF[x0] + 21 存入 RF[x20]`                           | `被跳过 RF[x20] = 0`                   |
| `78` | `blt x1 x12 +8`   | `比较 RF[x1] 和 RF[x12] 小于就跳转`                        | `大于 pc_next = pc + 4`               |
| `7C` | `addi x20 x0 22`  | `RF[x0] + 22 存入 RF[x20]`                           | `RF[x20] = 0x16`                    |
| `80` | `bge x12 x1 +8`   | `比较 RF[x12] 和 RF[x1] 大于等于就跳转`                      | `小于 pc_next = pc + 4`               |
| `84` | `addi x21 x0 23`  | `RF[x0] + 23 存入 RF[x21]`                           | `RF[x21] = 0x17`                    |
| `88` | `jal x22 +8`      | `跳转到 PC + 8 = 90 把 PC + 4 存入 x22`                  | `下一步执行 PC90 RF[x22] = 0x8C`         |
| `8C` | `addi x23 x0 24`  | `RF[x0] + 24 存入 RF[x23]`                           | `暂时被跳过? RF[x23] = 0`                |
| `90` | `addi x23 x0 25`  | `RF[x0] + 25 存入 RF[x23]`                           | `被执行 RF[x23] = 0x19`                |
| `94` | `addi x24 x0 169` | `RF[x0] + 169 存入 RF[x24]`                          | `RF[x24] = 0xa9`                    |
| `98` | `jalr x25 0(x24)` | `跳转到 RF[x24] + 0 清零后 = a8 将PC + 4 = 9C 写入 RF[x25]` | `RF [x25] = 9C next_pc = a8`        |
| `9C` | `addi x26 x0 1`   | `RF[x0] + 1 存入 RF[x26]`                            | `被跳过`                               |
| `A0` | `addi x26 x0 2`   | `RF[x0] +2 存入 RF[x26]`                             | `被跳过`                               |
| `A4` | `addi x26 x0 3`   | `RF[x0] + 3 存入 RF[x26]`                            | `被跳过`                               |
| `A8` | `addi x26 x0 26`  | `RF[x0] + 26 存入 RF[x26]`                           | `RF[x26] = 0x1a`                    |
![[b18cd8e632c4e4668281aa55088d316c.png]]
结合波形图 全对!!!!
**重点注意一下 shamt 用的是低五位 别搞错了**