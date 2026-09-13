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
