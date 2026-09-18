## 模块职责
---
ALU Decoder负责根据：
```
alu_op
funct3
funct7[5]
```
产生最终的：
`alu_control`
整体关系：
```
Opcode
   ↓
Main Decoder
   ↓
alu_op
   ↓
ALU Decoder ← funct3、funct7[5]
   ↓
alu_control
   ↓
ALU
```

## ALUOp
---
```
00 → 直接执行ADD
01 → 直接执行SUB
10 → 继续根据funct3和funct7[5]译码
```
## R-type译码
---

|funct3|funct7[5]|操作|
|---|---|---|
|`000`|0|ADD|
|`000`|1|SUB|
|`111`|X|AND|
|`110`|X|OR|
|`100`|X|XOR|

当前只需要输入：
```
funct7[5] = Instruction[30]
```
因为第一版支持的 R-type指令中，ADD与SUB需要通过该 Bit区分。
## ALUControl编码
---
```
0000 → ADD
0001 → SUB
0010 → AND
0011 → OR
0100 → XOR
```

这套编码必须与 ALU模块保持一致。

## 模块性质

ALU Decoder属于组合逻辑：

```
不保存State
不需要Clock
不需要Reset
```

核心结论：

Main Decoder判断指令大类，ALU Decoder判断具体ALU操作​