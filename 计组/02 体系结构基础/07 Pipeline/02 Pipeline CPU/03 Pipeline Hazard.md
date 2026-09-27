 真实程序在多条 Instruction 重叠执行后 可能出现
```
Hardware 不够用 -- Structural Conflict
Operand 尚未准备好 -- Data Hazard
正确 Next PC 尚未确定 -- Control Hazard
```
这就是 **Pipeline Hazard**

## Hazard
---
先看表示
```
如果 Pipeline 继续按照正常速度推进
某条 Instruction
就可能使用错误的 Hardware
错误的 Data
或错误的 Instruction Address
```
并不代表着 **CPU已经得到错误结果** 只是表示当前存在可能导致错误执行的条件
**只要 Microarchitecture 能够检测并正确处理 Hazard 最终 Architectural State 仍然符合 ISA**

所以存在
```
Instruction Dependency
↓
可能形成 Hazard
↓
Hazard Control 采取措施
↓
保证执行结果正确
```

## Structural Hazard
---
Structural Hazard 结构冒险 多个 **流水级** 在同一 Cycle 争用同一个硬件资源

产生的本质原因是
`同时发生的 Operation > Hardware 能提供的 Operation`
常见例子
```
IF 与 MEM 争用统一 Memory
两条 Instruction  争用单端口 Register File
多个执行单元 争用同一个乘法器
Instruction Fetch 与 Load 争用 Cache Port
```

## 处理 Structural Hazard
---
主要有两个方向
1. 增加或分离 Hardware Resource
2. 暂停其中一个 Operation

> 增加或分离 Hardware Resource

例如单端口Memory -> 双端口Memory 使得一个周期可以读取和访问同时进行

> 暂停其中一个 Operation

如果无法增加 Hardware 那么只能
`优先完成一个Operation 暂停另一个 Stage`
这种等待则称为
**Stall - Pipeline Stall - 流水线停顿**
但不是简单的停止整个CPU 需要有策略的停顿

## Data Hazard
---
Data Hazard 数据冒险 Instruction 之间的**数据依赖** 与 流水线**执行时序**发生冲突

Data Dependency 数据依赖 一条 Instruction 所需的数据由另一条 Instruction 产生

但是 **Data Dependency 不一定造成 Data Hazard**
而且 相同程序放到不同的 Pipeline中 可能出现不同的 Hazard

## RAW Data Hazard
---
即 Read After Write 写后读相关 这是基础顺序执行 Pipeline 中最重要的

含义为
```
前一条 Instruction 应该先写入某个位置
后一条 Instruction 随后读取同一位置
```

注意区分
`Result Available 结果有效` 和 `Result Committed to Register File 结果寄存`
在基础五级 Pipeline 中
```
ALU Instruction 的 Result
通常在 EX 末尾已经产生
但在 WB Stage
才写入 Register File
```
所以 有时 Data 其实已经存在于
`EX/MEM Register` 或 `MEM/WB Register`
只是还没有存到 `RF` 中
**这为后续 Forwarding 提供可能**

## Forwarding
---
Data Forwarding 数据前递
在 `Result` 已经在 Pipeline 中产生 但尚未写回 `RF` 时
直接将这个 `Result` 送到依赖该 `Instruction` 的 ALU Input

例如
```
ADD x3 x1 x2
SUB x4 x3 x5
```
可以有路径
```
ADD ALU Result
↓
EX/MEM Register
↓
Forwarding MUX
↓
SUB ALU Operand
```
**Forwarding 改变的是 Operand 的实际来源**
正常来源：
```ID/EX.rs1_data```
发生 Forwarding 时：
```EX/MEM.ALUResult```
或：
```MEM/WB.Result```
覆盖原来的 Operand

也会出现 rs1 和 rs2 同时需要 Forwarding 的情况
`x0` 通常被排除在 Hazard 外 因为 `x0` 不可写入 
在 Hazard 比较中 一般要求 `rd != 0`

## Load-Use Hazard
---
载入后 立即使用 冒险
例如
```
LW  x3 0(x1)
ADD x4 x3 x5
```
`ADD` 需要 `LW` 读出的 Memory Data

| Cycle | IF  | ID  | EX  | MEM | WB  |
| ----- | --- | --- | --- | --- | --- |
| 1     | LW  |     |     |     |     |
| 2     | ADD | LW  |     |     |     |
| 3     | I3  | ADD | LW  |     |     |
| 4     | I4  | I3  | ADD | LW  |     |
| 5     | I5  | I4  | I3  | ADD | LW  |
在 Cycle 3：
```
LW 位于 EX
只完成 Effective Address Calculation

ADD 位于 ID
准备进入 EX
```
到了 Cycle 4：
```
LW 位于 MEM
Memory Data 在这一 Stage 才产生

ADD 位于 EX
在同一个 Cycle 中需要这个 Data
```
对于常见五级 Pipeline：
```
LW 的 Data
到 MEM Stage 末尾才准备好

下一条 ADD 的 ALU
在 EX Stage 中需要 Operand
```
这个 Data 通常来不及直接满足紧邻的下一条 Instruction
即使有普通 Forwarding 仍然需要让 `ADD` 等待

## Load-Use 的解决
---
通过 `Stall` 可以解决 还是刚刚的例子
在 `ADD` 指令的执行过程中 插入一个 等待Cycle
```
Cycle 3
LW  → EX
ADD → ID

Cycle 4(对于ADD来说 这一Cycle在等)
LW  → MEM
Bubble → EX
ADD 保留在 ID

Cycle 5
LW  → WB
ADD → EX
```
`ADD` 在 Cycle 5 才执行EX 所需的 `Load Data` 可以从后级 `Forward` 过来
于是产生了
**Bubble 流水线气泡**
表示 `某个 Pipeline Stage 中 当前没有会修改 Architectual State 的有效 Instruction`
会占用一个 Stage 和一个 Cycle 但不产生 Register Write 等操作

## 区分 Stall 和 Bubble
---
`Stall = 某些 Pipeline State 暂停前进`
例如 Load-Use Hazard 时
`PC 保持 IF/ID 保持` 这样 `ADD` 停留在 `ID`

`Bubble = 向某个后续 Stage 注入无效 Operation`
例如 `ID/EX Control <- 0`
典型动作是：
```
PC        不更新
IF/ID     不更新
ID/EX     写入 Bubble
EX/MEM    正常前进
MEM/WB    正常前进
```

## WAR 与 WAW
---
除了 `RAW` 之外 理论上还有两种 Data Dependency
**WAR Write After Read 读后写相关
WAW Write And Write 写后写相关**

由于当前的 Pipeline
```
Instruction 按程序顺序进入
Register File Read 固定在 ID
Register File Write 固定在 WB
Instruction 按程序顺序到达 WB
```

所以 **WAR 不会形成实际 Hazard**
- 前一条 Instruction 一定先进入 ID 并读取 Operand
- 后一条 Instruction 才会到达 WB 并写入 Register File
因此不会出现：
```后一条先写 导致前一条读到新值```

**WAW 也不会形成实际 Hazard**
两条 Instruction 按程序顺序进入 Pipeline
也按程序顺序到达 WB
所以写入顺序不会颠倒
最终仍然是后一条 Instruction 的结果留下

所以 在我们现在的 **In-Order Pipeline** 中 这两个都不会形成实际 Hazard 只有 **RAW Hazard** 存在

## Control Hazard
---
控制冒险 即 Next PC 尚未确定时 后续 Instruction 已经计入 Pipeline 可能会导致CPU暂时走入错误路径 
例如 :

|Cycle|IF|ID|EX|MEM|WB|
|---|---|---|---|---|---|
|1|BEQ|||||
|2|ADD|BEQ||||
|3|SUB|ADD|BEQ|||
在 Cycle 3：
```
BEQ 在 EX
得到 Compare Result
计算或选择 Branch Target
```
同时：
```
ADD 已经进入 ID
SUB 已经进入 IF
```
如果 `BEQ Taken`：
```
ADD 错误
SUB 错误
```
它们不能继续修改 Architectural State

## Flush
---
即 流水线冲刷
当发现 某些已经进入 Pipeline 的 Instruction 来自错误路径时 **把这些 Instruction 标记为无效**

概念上：
```
BEQ 在 EX 确认 Taken
PC ← Branch Target
IF/ID  ← Invalid
ID/EX  ← Invalid
```
于是错误路径中的：
```
ADD
SUB
```
不会继续执行有效的：
```
Register Write
Memory Write
```
之后 IF Stage 从正确 Target 重新取指

Flush 的核心 
**不是删除已经发生过的组合计算 而是保证错误路径 Instruction 不能更新 Architectural State**

对于 `SW` Control Hazard 特别危险 如果不及时在产生副作用之前 清除Control 则会导致 Architectural State 被错误更改
关键副作用包括：
```
Register File Write
Data Memory Write
Control Register Write
I/O Write
```

## Branch Prediction
---
处理 Control Hazard 的一种方向 :
**提前 Branch Resolution** 
但是会使得前级 Hardware 和 Control 可能更复杂

于是我们可以采取 **Branch Prediction** 分支预测 在 Branch 真正解析前 预测控制流方向或目标
最简单的 就是预测 `Predict Not Taken`
然后顺序执行 最终结果
- `Not Taken` -> 预测正确 Pipeline不清楚
- `Taken` -> 预测错误 Flush错误路径 PC 转向 Branch Target

当前的基础Pipeline可以采用该模式

## Hazard Detection Logic
---
是组合逻辑 **观察 Pipeline Register 中保存的信息 检测各种Hazard**

> 检测 Data Hazard

可能比较：
```
Source Instruction.rs1
Source Instruction.rs2
Producer Instruction.rd
Producer Instruction.RegWrite
Producer Instruction.Result Type
```

> 检测 Load-Use Hazard

可能观察：
```
当前 EX Stage Instruction 是否是 LW
它的 rd 是什么
当前 ID Stage Instruction 是否读取这个 rd
```

> 检测 Control Hazard

可能观察：
```
Branch / Jump 是否成立
Target Address
当前哪些 Pipeline Register
包含错误路径 Instruction
```

## Hazard Control 实际控制
---
Hazard Control 最终必须作用于 State Element
常见控制包括：
```
PC Enable
IF/ID Enable
IF/ID Clear
ID/EX Enable
ID/EX Clear
EX/MEM Clear
Forwarding MUX Select
```

> Enable 和 Clear

`Enable = 0` 表示 Clock Edge 时 `Register 保持原值` 用于 **Stall**
`Clear = 1` 表示 Clock Edge 时 `Register 进入无效状态` 通常通过 `Control Signals 清零`实现 用于**插入 Bubble 或 Flush**

> MUX Select

决定 `Operand` 来自
```
原始 ID/EX Data
EX/MEM Result
MEM/WB Result
```
**用于 Forwarding**

Hazard Control 并不亲自计算 Instruction Result
只是决定
- Pipeline State 是否前进
- 某个 Instruction 是否仍然有效
- Operand 从哪里获得


## Valid Bit
---
有效位
例如
```
IF/ID.valid
ID/EX.valid
EX/MEM.valid
MEM/WB.valid
```
含义
```
valid = 1
该 Pipeline Register 中包含有效 Instruction
valid = 0
该位置是 Bubble 或无效内容
```

当无效 Instruction 到达 MEM 或 WB:
```
即使其他 Data Bit 中仍然有值
也不能产生 Architectural Side Effect
```
也可以不显式保存 `Valid Bit` 通过把全部副作用 Control 清零表示 Bubble 本质相近

## 同一 Cycle 可能存在多个 Hazard 判断
---
假设：
```
LW   x3 0(x1)
BEQ  x3 x4 target
```
第二条 Instruction：
```
依赖 LW 的 x3
同时又决定 Control Flow
```
这里同时涉及：
```
Data Hazard
Control Hazard
```
这说明 Hazard 并不是三个完全孤立的模块 他们可能在同一组 Instruction 上相互影响 控制逻辑需要确定正确的优先级 在基础设计中 通常需要保证
```
错误或尚未准备好的 Data
不能驱动 Architectural Decision
```