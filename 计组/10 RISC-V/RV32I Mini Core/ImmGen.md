## ImmGen职责
---
ImmGen — Immediate Generator — 立即数生成器。
负责根据 `imm_src`，从32-bit Instruction中提取并重新排列 Immediate，然后扩展为32 bit。
```
Instruction
imm_src
    ↓
  ImmGen
    ↓
32-bit Immediate
```
ImmGen属于组合逻辑：

```
不保存State
不需要Clock
不需要Reset
```

## imm_src编码

```
00 → I-type
01 → S-type
10 → B-type
11 → 未定义
```

`imm_src`由 Main Decoder根据 Opcode产生。

## I-type

```
imm[11:0] = instruction[31:20]
```

```
{{20{instruction[31]}}, instruction[31:20]}
```

用于：

```
addi
lw
```

## S-type

```
imm[11:5] = instruction[31:25]
imm[4:0]  = instruction[11:7]
```

```
{
    {20{instruction[31]}},
    instruction[31:25],
    instruction[11:7]
}
```

用于：

```
sw
```

## B-type

```
imm[12]   = instruction[31]
imm[11]   = instruction[7]
imm[10:5] = instruction[30:25]
imm[4:1]  = instruction[11:8]
imm[0]    = 0
```

```
{
    {19{instruction[31]}},
    instruction[31],
    instruction[7],
    instruction[30:25],
    instruction[11:8],
    1'b0
}
```

用于：

```
beq
```

核心关系：

Instruction Format不同→Immediate字段位置不同→ImmGen重新排列​