回顾现在的基础设计
```
五级 Pipeline：IF → ID → EX → MEM → WB
Branch / JAL / JALR 在 EX Stage 确定 Next PC
```

## Branch 在 EX 决定时的 Pipeline 状态
---
举例
```
BEQ x1 x2 target
ADD x3 x4 x5
SUB x6 x7 x8
```

时序

| Cycle | IF  | ID  | EX  | MEM | WB  |
| ----- | --- | --- | --- | --- | --- |
| 1     | BEQ |     |     |     |     |
| 2     | ADD | BEQ |     |     |     |
| 3     | SUB | ADD | BEQ |     |     |
在 关键的 Cycle 3 中
```
BEQ 在 EX
→ 比较 x1 与 x2
→ 产生 BranchTaken
→ 得到 BranchTarget

ADD 在 ID
→ 位于 ID/EX 的输入端

SUB 在 IF
→ 位于 IF/ID 的输入端
```

- 若 Branch Not Taken 
  `ADD` 和 `SUB` 都属于正确路径 Pipeline 正常前进
- 若 Branch Taken 
  则 `ADD` 和 `SUB` 都属于错误路径 必须取消

## Flush
---
流水线冲刷
**将已经进入 Pipeline 但不应该执行的 Instruction 变成无效**

在 `Branch Taken` 时 需要
```
PC ← BranchTarget
IF/ID ← Invalid
ID/EX ← Invalid
```

Flush 并不需要擦除所有的 `Data Bit`
可以采用两种实现方式
```
方法 1
valid ← 0
方法 2
所有副作用 Control ← 0
```

## Branch Taken 时的 Clock Edge
---
在 Clock Edge 之前
```
Current PC 已经指向顺序路径后续地址

EX Stage 产生 BranchTaken 与 BranchTarget
组合控制逻辑产生
PCSrc = BranchTaken
FlushIFID = BranchTaken
FlushIDEX = BranchTaken
```

到达 Clock Edge
```
PC ← BranchTarget
IF/ID ← Invalid
ID/EX ← Invalid
```

Clock Edge 之后
```
IF 从 BranchTarget 取指
EX 中出现 Bubble
ID 中出现 Bubble
```

## Branch Not Taken
---
当 `BranchTaken = 0` 时
之前取出的 Instruction 本来就是正确路径 不需要`Flush`

相当于采用了 `Predict Not Taken 预测分支不跳转`

## Branch Penalty
---
即分支惩罚
根据我们刚刚的分析 我们这套设计的 `Taken Branch Penalty` 通常是 `2 Cycles`
即有两个周期执行的是错误的 Instruction


## JAL 的 Flush
---
```
RF[rd] ← PC + 4
PC ← PC + Immediate
```
是无条件跳转 我们在 `EX Stage` 计算 `Jump Target`
```
JAL 到达 EX
↓
JumpTaken 必然为 1
↓
PC Redirect
↓
Flush IF/ID 与 ID/EX
```
必然会 `Redirect` 且 `PC + 4` 必须随着 Instruction 前进 在 `WB Stage` 写入 `rd`

## JALR 的 Flush
---
```
RF[rd] ← PC + 4
PC ← (RF[rs1] + Immediate) & ~1
```
也是无条件跳转 可以参考 JAL
但是需要 `rs1` 所以可能会出现 `Forwarding` 和 `Stall` 的需要

## Control Hazard Unit
---
对于 `Flush` `Control Hazard Logic` 通常产生
```
PCSrc / Redirect
FlushIFID
FlushIDEX
```

最终最终 `Control Hazard Logic` 的功能表

| State Element | 正常               | Stall  | Taken Branch / Jump |
| ------------- | ---------------- | ------ | ------------------- |
| PC            | Next PC          | 保持     | Target              |
| IF/ID         | 写入新取指            | 保持     | Clear               |
| ID/EX         | 写入 Decode Result | Bubble | Clear               |
| EX/MEM        | 正常前进             | 正常前进   | Branch 本身正常前进       |
| MEM/WB        | 正常前进             | 正常前进   | 正常前进                |
