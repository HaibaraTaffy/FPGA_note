在 `Data 还没准备好 Forwarding 也无法凭空产生 Data 时`
我们就需要 `Stall 流水线停顿` 和 `Bubble 流水线气泡`

## Stall
---
流水线停顿
**暂时阻止某些 Pipeline Register 更新 让某条 Instruction 停留在当前的 Stage**
例如
```
LW  x3 0(x1)
ADD x4 x3 x5
发生了 Load-Use Hazard
```
于是
```
ADD 暂时停留在 ID
LW 继续前进到 MEM 和 WB
```

## Bubble
---
流水线气泡
**某个 Pipeline Stage 中 当前没有有效 Instruction**

表示 `Bubble` 的两种方式
```
1. 显式 Valid = 0
2. 把所有副作用 Control Signal 清零 即Control Clear
例如
RegWrite = 0
MemWrite = 0
Branch   = 0
Jump     = 0
```

## Stall 和 Bubble 配合
---
 如果只有 `Stall` 会导致 `Consumer` 被重复执行
 如果只有 `Bubble` 会导致 `Consumer` 被覆盖

例如
```
LW  x3 0(x1)
ADD x4 x3 x5
```
正确操作
```
冻结 PC
冻结 IF/ID
向 ID/EX 注入 Bubble
```

## 时序比较
---
无 Stall 时

|Cycle|IF|ID|EX|MEM|WB|
|---|---|---|---|---|---|
|1|LW|||||
|2|ADD|LW||||
|3|I3|ADD|LW|||
|4|I4|I3|ADD|LW||
|5|I5|I4|I3|ADD|LW|
`ADD` 在 `EX` 需要的 Data 
`LW` 在 `MEM` 才能给出
**ADD 不能在 Cycle 4 正确执行**

插入 Bubble 后

| Cycle | IF  | ID  | EX     | MEM    | WB     |
| ----- | --- | --- | ------ | ------ | ------ |
| 1     | LW  |     |        |        |        |
| 2     | ADD | LW  |        |        |        |
| 3     | I3  | ADD | LW     |        |        |
| 4     | I3  | ADD | Bubble | LW     |        |
| 5     | I4  | I3  | ADD    | Bubble | LW     |
| 6     | I5  | I4  | I3     | ADD    | Bubble |
可以注意到 **Cycle 4** 的时候 `ADD` 保持在 `ID` 阶段 而 `LW` 已经进入 `MEM` 阶段
在 **Cycle 5** 时 `ADD` 进入 `EX` 阶段 而 `LW` 已经将所需要的 Data 准备好
此时可以通过 `Forwarding` 来获得正确的 Load Data

## Stall 时 不变的 State
---

| State Element | 动作        |
| ------------- | --------- |
| PC            | 保持        |
| IF/ID         | 保持        |
| ID/EX         | 写入 Bubble |
| EX/MEM        | 正常更新      |
| MEM/WB        | 正常更新      |
即 **Consumer 停留 Producer 继续向后**

## PC Enable IF/ID Enable
---
由于 `PC` 也需要保持功能 所以引入 `PC Enable`
在正常情况下 `PCEnable = 1`  Clock Edge 时 `PC <- Next PC`
发生 `Stall` 时 `PCEnable = 0` Clock Edge 时  `PC 保持原值`

同理 也需要引入`IF_ID_Enable`
在正常情况下 `IF_ID_Enable = 1` Clock Edge 时 `IF/ID <- 当前 Fetch Result`
当发生 `Stall` 时 `IF_ID_Enable = 0` Clock Edge 时 `IF/ID 保持原值`

## ID/EX Clear
---
对于 `ID/EX` `Stall` 时并不是保持
而是 `ID_EX_Clear = 1`Clcok Edge 时 `ID/EX <- Bubble`
即
```
ID/EX.valid ← 0
或
ID/EX.RegWrite  ← 0
ID/EX.MemWrite  ← 0
ID/EX.Branch    ← 0
ID/EX.Jump      ← 0
```

## Stall Control
---
`Hazard Detection Unit` 中的 Stall Control 
负责观察
```
ID/EX.MemRead
ID/EX.rd
IF/ID.rs1
IF/ID.rs2
```
基本条件
```
ID/EX 是 Load //通过 MemRead 来判断
且 ID/EX.rd != 0
且 ID/EX.rd 等于 IF/ID 的有效 Source Register
```

因为不是每个 Instruction 都具有 rs1 rs2 有可能有些没有 有些被立即数替代了

|Instruction|Uses rs1|Uses rs2|主要用途|
|---|---|---|---|
|R-Type|是|是|ALU Operand|
|ADDI|是|否|ALU Operand|
|LW|是|否|Address Base|
|SW|是|是|Address Base + Store Data|
|Branch|是|是|Compare|
|JAL|否|否|PC + Immediate|
|JALR|是|否|Jump Target|如果

如果无条件比较`ID/EX.rd` 和 `IF/ID.rs1` 或 `IF/ID.rs2` 那么可能会出现错误的 `Stall`
**需要结合当前 Instruction 类型来进行判断**

## 实际分析 Stall
---
由于 `RF` 的设计不同 会影响 `Stall` 需要几个Cycle
所以不能脱离具体 RD 和 Clcoking 假设讨论 Stall数量

我们现在采用 `RF` 组合读 + Clcok Edge 写入
在这种假设下 我们通过
`Forwarding + Load-Use Bubble` 可以解决五级 Pipeline 中的典型情况

## 区分于 Flush
---
**Stall = 等待正确 Instruction
Flush = 取消错误 Instruction**

如果二者同时出现 通常是
`Flush 错误路径 > 对错误路径做 Stall`
一般的控制优先级可以理解为
```
Reset
↓
Flush / Redirect
↓
Stall
↓
Normal Advance
```

原则是 `不能让错误路径 Instruction 因为 Stall 被错误地保留下来`

## Stall 对性能的影响
---
保证了正确性 但是增加了 CPI

## 目前为止的 Hazard 处理关系
---
```
Producer Data Timing
↓
判断是否可 Forward
↓
如果不可及时 Forward
↓
Stall Consumer
↓
向 EX 注入 Bubble
↓
Producer 继续前进
↓
Data 可用后再 Forward
```
