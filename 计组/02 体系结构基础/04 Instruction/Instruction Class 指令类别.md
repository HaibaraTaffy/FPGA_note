RV32I 中 我们目前重点认识：
```
Arithmetic 算术
Logic 逻辑
Shift 移位
Load 载入
Store 存储
Branch 条件分支
Jump 跳转
```

| 类别                   | RISC-V 示例 | 主要数据来源                                    | 主要更新目标        |
| -------------------- | --------- | ----------------------------------------- | ------------- |
| Arithmetic           | `add`     | Register + Register                       | Register      |
| Arithmetic Immediate | `addi`    | Register + Immediate                      | Register      |
| Logic                | `and`     | Register + Register                       | Register      |
| Shift                | `sll`     | Register + Shift Amount                   | Register      |
| Load                 | `lw`      | Base Register + Immediate + Memory        | Register      |
| Store                | `sw`      | Base Register + Data Register + Immediate | Memory        |
| Branch               | `beq`     | Register + Register                       | PC            |
| Jump                 | `jal`     | PC + Immediate                            | Register + PC |

## Arithmetic Instruction
---
算术指令 例如
```
add  x5, x6, x7
sub  x5, x6, x7
addi x5, x6, 10
```
下面解释均用这些例子

### ADD 加法
---
`add x5,x6,x7`
语义 : `RF[x5] <- RF[x6] + RF[x7]`
其中
```
rs1 = x6
rs2 = x7
rd  = x5
```

### SUB 减法
---
`sub x5, x6, x7`
语义 : `RF[x5] <- RF[x6] - RF[x7]`

### ADDI 立即数加法
---
`addi x5, x6, 10`
语义 : `RF[x5] <- RF[x6] + 10`
**imm只能写在后面**
## Logic Instruction
---
逻辑指令 例如
```
and x5, x6, x7
or  x5, x6, x7
xor x5, x6, x7 异或
```

语义 : 
`RF[x5] <- RF[x6] AND RF[x7]`
`RF[x5] <- RF[x6] OR RF[x7]`
`RF[x5] <- RF[x6] XOR RF[x7]`

## Immediate Logic Instruction
---
即 含有立即数的 逻辑指令
```
andi
ori
xori
```

## Shift Instruction
---
移位指令
RISC-V中 常见的 Register Shift
```
sll 逻辑左移
srl 逻辑右移
sra 算术右移
```

### SLL
---
例如：
```
sll x5, x6, x7
```
被移位的数据来自：`RF[x6]`
Shift Amount 来自 `RF[x7]` 的相关低位。
最终结果写入：
```
x5
```

### SRL
---
逻辑右移 左侧补0

### SRA
---
算术右移 左侧补原符号位

## Immediate Shift
---
含有立即数的 移位指令 **一般 Shift Amount 是立即数**
```
slli
srli
srai
```
例如 `slli x5, x6, 3`

## LOAD
---
载入 即从 `Memory -> Register`
例如
```
lw x5 8(x6)
```

- `LW - Load Word 载入字` 在RV32I 中 一个Word = 32bit
- `x6` 在这里是 **Base Register 基址寄存器** 存放Base Address 基地址
  CPU读取 `RF[x6] 即基地址` 作为地址计算的基础
- `8` 在这里是 **Offset 偏移量** 指相对于基地址 偏移了多少
- `EA` 即`Effective Address` 有效地址 由
  `EA = Base Address + Offset` 算出 在这里实际上是
  `EA = RF[x6] + 8`

完整语义推导 : 读取 `Memory[EA]` 把读出的数据写入`RF[x5]`
即 `RF[x5] <- Memory[RF[x6] + 8]`
**真正写入x5的数据 来自Memory 而不是x6**

由于EA的计算 需要涉及加法 所以 虽然`lw` 是Memory Instruction 但是可以使用 ALU 或 Adder 完成地址计算

## Store
---
存储 即从 `Register -> Memory`
例如
```
sw x5, 8(x6)
```
- `SW - Store Word 存储字`
其余可以参考`lw`
完整语义 : `Memory[RF[x6] + 8] <- RF[x5]`

**该条指令没有 rd 因为最终写入的是 Memory**

## Load/Store Architecture
---
载入 存储体系结构
**特点 普通数据的 Memory Access 由 Load/Store 指令完成**
意思是 普通数据需要先经过 Load/Store 来进入或离开 RF 与 Memory交互
计算指令 只负责 在RF层面进行存取

达到了 **Memory Access 与 Arithmetic Operation 在 ISA层面被明确分开**

## Branch
---
条件分支 **根据条件 决定 Next PC**
例如 **BEQ - Branch if Equal 相等则分支**
```
beq x5, x6, target
```

语义 : CPU比较 `RF[x5]` 和 `RF[x6]` 若相等 则
`BranchTarget -> PC` 否则 按原来递增 `PC + 4 -> PC`

用于 改变程序的 Control Flow
`Branch Target = PC + Branch Offset`
如果成立 则称为 `Branch Taken` 分支成立/采用分支

## Jump
---
跳转 直接改变 Control Flow
RISC - V 中 重要的
```
jal
jalr
```

## JAL
---
即 Jump And Link 跳转并保存返回地址
例如
```
jal x1, target
```
分两步 :
1. `RF[x1] <- PC + 4` 保存返回地址
2. `PC <- target` 跳转到目标地址

这里可以看到 **一条 Instruction 可以同时更新多个处理器状态**

## LUI
---
Load Upper Immediate 即加载高位立即数
例如:
```
lui x5, 0x12345
```

语义 `RF[x5] = 0x1234_5000`

## AUIPC
---
Add Upper Immediate to PC 即高位立即数与PC相加
语义为 `RF[rd] <- CurrentPC + UpperImmediate`


## Instruction 对应硬件行为
---
现在比较几条典型指令。
### ADD
```
add x5, x6, x7
```
需要：
```
Register File Read
→ ALU ADD
→ Register File Write
```
---
### ADDI
```
addi x5, x6, 10
```
需要：
```
Register File Read
→ Immediate Generator
→ MUX
→ ALU ADD
→ Register File Write
```
---
### LW
```
lw x5, 8(x6)
```
需要：
```
Register File Read
→ Immediate Generator
→ Address Calculation 地址计算器
→ Memory Read 存储器读
→ Register File Write
```
---
### SW
```
sw x5, 8(x6)
```
需要：
```
Register File Read
→ Immediate Generator
→ Address Calculation
→ Memory Write
```
---
### BEQ
```
beq x5, x6, target
```
需要：
```
Register File Read
→ Compare
→ Branch Target Calculation
→ Next-PC Selection
```

## 复用
---
由于 **不同 Instruction 会让同一套 Datapath 形成不同的数据流**
所以 可以通过 `Mux + Control` 来选择不同的 Operand 来源 复用同一个ALU

## 分析Instruction
---
分析一条 RISC-V Instruction 可以先看四个方面
1. RF 是否读取/写入
2. Execution Hardware 需要哪些?
3. Memory 是否读取/写入
4. Next PC 是递增/跳转?

一般 **一条 ISA Instruction 可以对应多个内部硬件操作**

分析流程:
1. 这条Instruction做什么?
2. 读取哪些Register
3. 真正数据来自哪里
4. 最终更新什么状态
5. 需要哪些硬件
6. Next PC是什么

## Micro-Operation 微操作
---
指处理器内部 一次比较基础的数据传送 运算 或者 状态更新
例如 `lw` 可以从功能上拆成：
```
Register Read
Address Calculation
Memory Read
Register Write
```

但是注意 `Micro-Operation ≠ MicroInstruction`
**MicroInstruction - 微指令 是某些控制器具体实现方式中的概念**
Micro-Operation 是更一般的处理器内部操作概念

## Instruction Width 和 Data Width
---
在RV32I中 整数 Register Width 和 基础 Instruction Width 均为 `32bit`
但是概念不同 只是数值恰好相同