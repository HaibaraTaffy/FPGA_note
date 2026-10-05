## 模块职责
---
Main Decoder — 主译码器。
负责读取：
`Instruction[6:0] = Opcode`
判断当前 Instruction属于哪一大类，然后产生主要 Control Signals：
```
Opcode
   ↓
Main Decoder
   │
   ├─ reg_write
   ├─ alu_src
   ├─ mem_write
   ├─ result_src
   ├─ branch
   ├─ imm_src
   └─ alu_op
```
Main Decoder只识别指令大类，不负责判断 R-type内部具体执行 ADD、SUB、AND、OR还是XOR。

## 第一版Opcode
---

| 类别     | Opcode    | 指令                           |
| ------ | --------- | ---------------------------- |
| OP     | `0110011` | `add`、`sub`、`and`、`or`、`xor` |
| OP-IMM | `0010011` | `addi`                       |
| LOAD   | `0000011` | `lw`                         |
| STORE  | `0100011` | `sw`                         |
| BRANCH | `1100011` | `beq`                        |

Opcode由 RISC-V ISA规定。

## Control Signals
---
```
reg_write
0 → 禁止写Register File
1 → 允许写Register File
```

```
alu_src
0 → Operand B选择RF[rs2]
1 → Operand B选择Immediate
```

```
mem_write
0 → 不写Data Memory
1 → 写Data Memory
```

```
result_src
0 → ALU Result写回
1 → Memory Read Data写回
```

```
branch
0 → 非条件Branch
1 → 条件Branch
```

```
imm_src
00 → I-type
01 → S-type
10 → B-type
```

```
alu_op
00 → ALU执行ADD
01 → ALU执行SUB，用于Branch比较
10 → 根据funct3和funct7继续译码
```

## Control Table
---

|类别|reg_write|alu_src|mem_write|result_src|branch|imm_src|alu_op|
|---|---|---|---|---|---|---|---|
|OP|1|0|0|0|0|00|10|
|OP-IMM|1|1|0|0|0|00|00|
|LOAD|1|1|0|1|0|00|00|
|STORE|0|1|1|0|0|01|00|
|BRANCH|0|0|0|0|1|10|01|
|未识别|0|0|0|0|0|00|00|

对于无实际作用的 Control Signal，第一版仍然给出确定的安全值。

## 组合逻辑结构
---
Main Decoder属于组合逻辑：
```
Opcode变化
    ↓
组合译码
    ↓
Control Signals变化
```
它不需要：
```
Clock
Reset
State Storage
```
RTL中先给所有输出默认值，再由 `case`覆盖当前指令需要的控制信号。
这样可以：
```
避免Latch
简化分支代码
为未识别Opcode提供安全值
```

## Main Decoder与ALU Decoder
---

```
Opcode
   ↓
Main Decoder
   ↓
alu_op
   ↓
ALU Decoder
   ↓
alu_control
   ↓
ALU
```

必须区分：
`alu_op ≠ alu_control`
`alu_op`只是运算类别；`alu_control`才是最终送给 ALU的操作选择信号。

## 核心结论
---

**Main Decoder根据Opcode配置整条Datapath​**