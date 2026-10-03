即数据前递 解决 
```
某个 Result 已经产生 但还没有到达正常存储位置 后续 Hardware 又立即需要这个 Result 
能否建立更短的数据路径 直接把 Result 送给 Consumer
```
## Forwarding 本质
---
Forwarding 增加旁路
```
EX/MEM Result ───────────┐
                        │
MEM/WB Result ───────────┼→ Forwarding MUX → ALU Input
                        │
ID/EX Register Data ─────┘
```
本质就是
**检测 Operand 是否已经过期 如果过期 寻找 Pipeline 后级中更新的数据 通过 MUX 选择最新的 Result**
只是改变当前 EX Stage 实际使用的数据来源

## Producer 和 Consumer
---
Producer 数据生产者 产生某个 Register 新值的 Instruction
Consumer 数据消费者 读取这个 Register 的后续 Instruction
**都是描述 Instruction 的名词**

于是 Forwarding Logic 需要回答 :
```
当前 EX Stage 的 Consumer 所需要的 rs1 和 rs2 
是否由 Pipeline 后级中的 Producer 产生
```

## Forwarding Unit
---
对于 `Consumer` 一般位于 `EX Stage` 时 `Forwarding Unit` 就要开始观察了
```
Consumer 端
ID/EX.rs1
ID/EX.rs2

Producer 端
EX/MEM.rd
EX/MEM.RegWrite
EX/MEM Result 是否已经准备好

MEM/WB.rd
MEM/WB.RegWrite
MEM/WB.Result
```
如何判断是否需要 Forwarding
```
判断地址相同?
ID/EX.rs1 == EX/MEM.rd
ID/EX.rs2 == EX/MEM.rd
ID/EX.rs1 == MEM/WB.rd
ID/EX.rs2 == MEM/WB.rd
还要满足
Producer 确实写 Register File RegWrite?
rd 不是 x0 
所需 Result 已经准备好
```

## Forwarding Control
---
对于 ALU 有两个 Operand 于是需要两个独立的 Forwarding Control
`ForwardA ForwardB`
分别用于控制 Operand 的来源
```
例如 ForwardA
00 → ID/EX.rs1_data
10 → EX/MEM Result
01 → MEM/WB Result
```

## EX/MEM Forwarding
---
以
```
ADD x3 x1 x2
SUB x4 x3 x5
```
为例
当 `SUB` 位于 `EX` 时 有`ID/EX.rs1 = x3`
此时 `ADD` 位于 `MEM`
```
EX/MEM.rd = x3
EX/MEM.RegWrite = 1
EX/MEM.ALUResult = 新的 x3
```
检测匹配条件
```
EX/MEM.RegWrite = 1
EX/MEM.rd != 0
EX/MEM.rd == ID/EX.rs1
```
所以需要进行 `Forwarding` 令 `ForwardA = EX/MEM`
实际的数据流
```
EX/MEM.ALUResult
↓
Operand A Forwarding MUX
↓
SUB ALU Operand A
```
此时 另一个 Operand 不发生`Forward`
```
ID/EX.rs2_data
↓
SUB ALU Operand B
```
最终可以执行 `最新的x3 - x5`

## MEM/WB Forwarding
---
以
```
ADD x3 x1 x2
OR  x8 x6 x7
SUB x4 x3 x5
```
为例 其实和 `EX/MEM Forwarding` 差不多 只是这个时候 `ADD` 信息在 `MEM/WB`
检测条件
```
MEM/WB.RegWrite = 1
MEM/WB.rd != 0
MEM/WB.rd == ID/EX.rs1
```
这里最好使用 最终的`Write-Back Result` 因为 `ALUResult` 可能不是最终写入 `RF` 的数据
数据路径
```
MEM/WB.Result
↓
Forwarding MUX
↓
SUB ALU Operand A
```

## 两个 Operand 同时 Forward
---
例如
```
ADD x3 x1 x2
SUB x4 x3 x3
```
`SUB` 的 两个 Operand 都依赖 `ADD` 发生了同时 `Forward`
各自判断 执行`Forward` 即可

## 两个 Operand 来自不同 Producer
---
例如
```
ADD x3 x1 x2
OR  x4 x5 x6
SUB x7 x4 x3
```
对于 `SUB` 需要
```
ForwardA = EX/MEM
ForwardB = MEM/WB
```
同一个 ALU 的两个输入 需要在同一 Cycle 从不同的 Pipeline Stage 获得数据
```
EX/MEM.OR Result
↓
Operand A

MEM/WB.ADD Result
↓
Operand B
```
**ForwardA 与 ForwardB 必须完全独立**

## 多个 Producer 同时匹配
---
一般来说 在多个 Producer 同时匹配时 例如
```
ADDI x3 x0 1
ADDI x3 x3 1
ADD  x4 x3 x5
```
会选择距离 `Consumer` 最近的 `Producer` 即读取第二条产生的x3
选择`EX/MEM Result`
本质是 **距离 Consumer 最近的 Producer 代表 Program Order 中最近的一次有效写入**

## Forwarding 优先级
---
- 首先检查 `EX/MEM`
- 其次检查 `MEM/WB`
- 否则使用 `ID/EX` 原始 Operand

## Forwarding MUX 需要位于哪里
---
先明确一点 `Forwarding` 不是为 `ALU` 专门设计 也会出现 `Store Write Data` 需要 `Forwarding` 的情况 
于是 对于 `rs1`
```
ID/EX.rs1_data ───┐
EX/MEM Result ─────┼→ ForwardA MUX → ALU Operand A
MEM/WB Result ─────┘
```
对于 `rs2`
```
ID/EX.rs2_data
↓
Forwarding MUX
↓
ForwardedB
├────────────→ Store Data // 在store指令里用的
│
└→ ALUSrc MUX //在 ALU 指令里用的
   ├── ForwardedB
   └── Immediate
        ↓
      ALU Operand B
```

## Store Data Hazard
---
以
```
ADD x3 x1 x2
SW x3 0(x4)
```
为例

对于`SW` 我们必须知道
**rs1 = x4 rs2 = x3**
因为 `ADD` 产生 新的 `x3`
而 `SW` 到达 `MEM` 的时候 必须使用新的 `x3`
所以我们需要建立 `Store Data Forwarding`
采取
```
ADD Result
↓
Forwarding MUX
↓
SW 的 Store Data Path
↓
EX/MEM.WriteData
↓
Data Memory
```
更新 `EX/MEM.WriteData` 否则可能读取到 `SW Decode` 阶段读取的旧值 (WriteData 读取之后就传下来了 直到更新)

## Branch Operand Forwarding
---
例如
```
ADD x3 x1 x2
BEQ x3 x4 target
```
BEQ 的比较 Operand 可以使用相同的 Forwarding MUX
```
EX/MEM.ADD Result
↓
ForwardA MUX
↓
Branch Comparator 或 ALU
```
我们的设计保持 `Branch 在 EX Stage 解析`

## JALR 的 Operand Forwarding
---
例如
```
ADD  x3 x1 x2
JALR x5 0(x3)
```
JALR 在 `EX Stage` 计算 Target
可以复用
```
EX/MEM Result
↓
ForwardA MUX
↓
JALR Target Calculation
```

## Load Result 特殊
---
例如
```
LW  x3 0(x1)
ADD x4 x3 x5
```
`ADD` 位于 `EX` 时
`LW` 位于 `MEM`
此时 `EX/MEM` 中保存的并不是 `Memory Read Data` 而是 `Effective Address`
会出现错误

所以 对于`EX/MEM Forwarding` 还必须确认
**Producer 的 最终 Register Result 是否已经在这个位置准备好**

对于我们的 基础五级 Pipeline
```
R-Type Result
EX Stage 末尾准备好
I-Type ALU Result
EX Stage 末尾准备好
LW Result
MEM Stage 末尾才准备好
```
于是牵扯出 `Result Availability`

## Result Availability
---
是否能 `Forward` 需要看 `Result` 是否已经产生可用

| Producer      | Result 内容   | 最早产生位置 | 紧邻 Consumer 能否直接使用   |
| ------------- | ----------- | ------ | -------------------- |
| ADD SUB AND 等 | ALU Result  | EX 末尾  | 通常可以 EX/MEM Forward  |
| ADDI          | ALU Result  | EX 末尾  | 通常可以 EX/MEM Forward  |
| LW            | Memory Data | MEM 末尾 | 通常需要 Stall           |
| JAL           | PC + 4      | 较早已知   | 取决于 Datapath 是否提供该路径 |
| JALR          | PC + 4      | 较早已知   | 取决于 Datapath 是否提供该路径 |
|               |             |        |                      |
解决 Result 尚未及时产生的问题 需要`Stall`

## Load 到 Store 的特殊情况
---
例如
```
LW x3 0(x1)
SW x3 0(x4)
```
时间关系
```
LW 在 MEM 时
产生 Load Data
SW 同时在 EX
计算自己的 Store Address
下一 Cycle
SW 才在 MEM 使用 Store Data
```
如果 Datapath 提供：
```
MEM/WB.Load Result
↓
Data Memory Write Data
```
那么 SW 可以在自己的 MEM Stage 获得 Load Data
避免了 Stall

## Forwarding 优先级表
---

|条件|Operand 来源|
|---|---|
|EX/MEM 是有效 Producer 且 rd 匹配|EX/MEM Result|
|EX/MEM 不匹配 但 MEM/WB 是有效 Producer 且 rd 匹配|MEM/WB Result|
|都不匹配|ID/EX Register Data|
有效 `Producer` 至少要求
```
valid = 1
RegWrite = 1
rd != x0
Result 已准备好
```

其中 被 `Flush` 的 Instruction 通过 
`RegWrite` 和 `Valid` 来保证 无效Instruction 不产生影响

## Forwarding 对 Critical Path 的影响
---
Forwarding 提高性能的方式 `减少 Stall`
但是同时会增加 Hardware 可能会增加延迟
形成一个 `Trade-Off`

## 核心原则
---
**Result 的逻辑归属位置 不一定是 Consumer获取它的唯一来源 只要 Data 已经产生 Microarchitecture 可以建立更直接的传递路径**

