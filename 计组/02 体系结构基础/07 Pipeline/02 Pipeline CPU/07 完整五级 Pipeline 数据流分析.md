## 当前设计
---
- Pipeline `IF → ID → EX → MEM → WB`
- Pipeline Register ```IF/ID ID/EX EX/MEM MEM/WB```

基本假设
```
Instruction Memory 与 Data Memory 分离
Register File 组合读取
Register File 在 Clock Edge 写入
ALU Result 在 EX Stage 末尾产生
Load Data 在 MEM Stage 末尾产生
Branch / JAL / JALR 在 EX Stage Redirect PC
ALU Operand 支持 EX/MEM 与 MEM/WB Forwarding
Load-Use Hazard 插入一个 Bubble
Branch 采取 Predict Not Taken
```

## Pipeline Register 设计
---
> IF/ID

```
valid
Instruction
PC
PC + 4
```

> ID/EX

```
valid
rs1_data
rs2_data
rs1
rs2
rd
Immediate
PC
PC + 4
EX Control
MEM Control
WB Control
```

> EX/MEM

```
valid
ALU Result
Store Data
Branch / Jump Target
Branch Condition
rd
PC + 4
MEM Control
WB Control
```

> MEM/WB

```
valid
ALU Result
Memory Read Data
PC + 4
rd
WB Control
```

**原则**
```
Data 
Register Address 
Control 
Valid Information
```
**必须属于同一条 Instruction 并在 Pipeline 中保持对齐**

## Hardware 和 Stage 对应
---
在 Clcok Edge 之前
```
I1 位于 MEM
I2 位于 EX
I3 位于 ID
I4 位于 IF
```
所占用的 `Hardware`
```
I1 使用 Data Memory
I2 使用 ALU
I3 使用 Decoder Register File ImmGen
I4 使用 Instruction Memory
```

Clcok Edge 到达 所有 Register 同时更新
```
MEM/WB ← I1 的 MEM Result
EX/MEM ← I2 的 EX Result
ID/EX  ← I3 的 Decode Result
IF/ID  ← I4 的 Fetch Result
PC     ← Next PC
```

Clock Edge 之后
```
I1 进入 WB
I2 进入 MEM
I3 进入 EX
I4 进入 ID
```

## 不同的情况
---
1. 完全独立 Instruction `正常Pipeline前进`
2. EX/MEM Forwarding `Forwarding`
3. MEM/WB Forwarding `Forwarding`
4. 两个 Operand 来自不同位置 `分别 Forwarding`
5. 多个 Producer `先选择EX/MEM 不匹配再选 MEM/WB 否则 选择 ID/EX(原来的)`
6. Load-Use Stall `Stall + Forwarding`
7. 连续 Data Dependency `看清依赖关系 插入 Stall`
8. Store Address Forwarding `Forwarding`
9. Store Data Forwarding `Forwarding`
10. Branch Not Taken `正常 Pipeline 前进`
11. Branch Taken `Flush 后 产生 Bubble`
12. Branch Operand Forwarding `Forwarding`
13. Load 后紧跟 Branch `Stall 后可能需要 Flush`
14. JAL `直接Flush`
15. JALR 与 Forwarding `Forwarding后 Flush`

## Instruction Age
---
指令年龄
即 **Instruction 在 Program Order 中相对先后的关系**
在我们的顺序五级 Pipeline 中
`越靠近WB越老 越靠近IF越年轻`

Hazard Control 通常需要根据 Age 决定
```
哪个 Producer 更新
哪个 Instruction 应该等待
哪个 Instruction 应该被 Flush
哪个 Side Effect 应该保留
```

例如
```
`多个 Producer 同时匹配
→ 选择离 Consumer 最近的较年轻 Producer
`Branch Redirect
→ Flush Branch 后面的年轻 Instruction
`更老 Instruction
→ 继续 Commit
```

## 完成边界
---
不同 Instruction 修改 Architectual State 的位置不同
```
ALU Instruction
→ WB 写 Register File

LW
→ WB 写 Register File

SW
→ MEM 写 Data Memory

Branch
→ EX 决定 PC Redirect

JAL / JALR
→ EX Redirect PC
→ WB 写 Link Register
```
所以 `Flush` 要及时 不能等到已经完成了才执行 `Flush`

## Valid 引入
---
可以通过 `Valid` 信号 来控制 `Instruction` 和 `Pipeline Register` 是否有效

只有`valid = 1` 并且对应的 Control 有效 才能修改 Architectual State

## Forwarding Unit
---
**负责产生 Forward 请求**
观察
```
ID/EX.rs1
ID/EX.rs2
EX/MEM.rd
MEM/WB.rd
RegWrite
valid
Result Availability
```
产生
```
ForwardA
ForwardB
Store Data Forwarding Select
```

## Hazard Detection Unit
---
**负责产生 Stall 请求**
观察
```
ID/EX 是否为 Load
ID/EX.rd
IF/ID.rs1
IF/ID.rs2
UsesRs1
UsesRs2
```
产生
```
StallRequest
```

## Control-Flow Logic
---
**负责解决 Redirect**
观察
```
Branch
Jump
funct3
Forwarded Compare Operand
Target
```
产生
```
RedirectRequest
RedirectTarget
```

## Pipeline Control
---
**负责最后的总控制**
组合所有的请求
```
Reset
RedirectRequest
StallRequest
```
控制
```
PC Enable / Select
IF/ID Enable / Clear
ID/EX Clear
Pipeline Valid
```

一般的优先级
```
如果 Reset
→ 清空 Pipeline
→ PC 返回 Reset Address

否则如果 Redirect
→ PC 写入 Target
→ Flush 年轻 Instruction

否则如果 Stall
→ PC 保持
→ IF/ID 保持
→ ID/EX 注入 Bubble

否则
→ 所有 Pipeline Stage 正常前进
```