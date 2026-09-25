虽然 Multi-Cycle 已经解决了 Instruction 和 Cycle 绑定的问题 但是并没有解决另一个问题
**虽然每个阶段使用的 Hardware 不同
但同一时刻只有一条 Instruction 在执行**
很多 Hardware 在部分 Cycle 中仍然处于空闲状态
于是出现 Pipeline CPU 解决
**如何让多条 Instruction 同时处于不同的执行阶段**

## Pipeline 基本思想
---
即 `Instruction Pipeline` 指令流水线
**Pipeline 将 Instruction 的执行过程划分为多个 Stage
并允许不同 Instruction 同时位于不同 Stage**

```
Cycle 1    A：F
Cycle 2    A：D    B：F
Cycle 3    A：E    B：D    C：F
Cycle 4    A：M    B：E    C：D    D：F
Cycle 5    A：W    B：M    C：E    D：D    E：F
```
在 Cycle 4 的时候
```
Instruction A 正在 Memory 阶段
Instruction B 正在 Execute 阶段
Instruction C 正在 Decode 阶段
Instruction D 正在 Fetch 阶段
```
这就是大名鼎鼎的 `Instruction Overlap` 指令重叠执行
多条指令同时处于不同执行阶段

**Pipeline 并不会减少 单条 Instruction 的 Latency 而是在 Pipeline 填满之后 理想情况下 每个Cycle 都可以完成一个Instruction 提高了 Throughout**

## Pipeline Stage
---
流水级 是指令执行过程中的一个分段
一般每个 Stage 包含
`Combinational Hardware + Stage Register`
例如:
```
IF Stage
↓
IF/ID Register
↓
ID Stage
↓
ID/EX Register
↓
EX Stage
↓
EX/MEM Register
↓
MEM Stage
↓
MEM/WB Register
↓
WB
```

职责
```
保存某条 Instruction 在当前阶段产生的结果
(数据和控制 都需要被保存)
隔离相邻 Stage 的组合逻辑
```

## 常见的五级 Pipeline
---
经典 RISC 处理器经常使用五级 Pipeline：
```IF ID EX MEM WB```

> IF Instruction Fetch 取指

```
使用PC读取Instruction
计算顺序执行地址
保存取出的 Instruction 和相关 PC 信息
```

典型数据
`PC Instruction PC + 4`

> ID Instruction Decode 指令译码

```
解析 Instruction Fields
读取 Register File
生成 Immediate
识别 Instruction Type
准备 Control Information
```

典型数据
`rs1/rs2_data Immediate rd funct3 funct7 Control Signals`

> EX Execute 执行

**主要工作其实是取决于 Instruction Type**
```
ALU Operation 算术操作
Memory Address Calculate 存储器地址计算
Branch Compare 分支比较
Branch Target Calculate 分支计算
```

>MEM Memory Access 存储器访问

主要处理 `LW Memory Read` 和 `SW Memory Write`
**但是对于普通的算术 Instruction 虽然不访问 但是依旧需要让 Instruction 流下去**

>WB Write Back 写回

```
ALU Result 写回 Register File
Memory Read Data 写回 Register File
```
**不同 Instruction 选择不同 WB 来源**

## Data Hazard
---
当后一条 Instruction 可能依赖前一条 Instruction 的结果时 例如
```
ADD x3 x1 x2
SUB x4 x3 x5
```
`SUB` 需要 `x3` 但 `ADD` 并未完成 `WB` 所以 `SUB` 可能读取旧值 即  
**Data Hazard 数据冒险**
**Instruction 之间存在Data Dependency 而导致执行冲突

## Control Hazard
---
若 `Branch` 的结果还没有确定 而后续的 Instruction 就已经被 `FETCH` 导致指令混乱 被称为
**Control Hazard 控制冒险**
**控制流改变 导致已进入 Pipeline 的 Instruction 可能无效**

## Structural Conflict
---
不同的 Instruction 在同一个 Cycle 中 需要同一个 Hardware
例如：
```
Instruction A 需要访问 Data Memory
Instruction B 需要 Fetch Instruction
```
如果 Instruction Memory 和 Data 共用 单端口Memory 那么二者会发生资源冲突 即
**Structural Hazard 结构冒险**
**多条 Instruction 同时需要同一硬件资源**

## Pipeline 收益与代价
---
Pipeline 的收益：
```
提高 Instruction Throughput
提高 Hardware 利用率
缩短单个 Stage 的组合路径
允许更高的 Clock Frequency
```

Pipeline 的代价：
```
增加 Pipeline Register
增加 Control 复杂度
增加数据传递路径
出现 Data Hazard
出现 Control Hazard
可能出现 Structural Hazard
需要 Stall、Forwarding、Flush 等机制
```

## Pipeline Fill 和 Pipeline Drain
---
>填充阶段

刚开始工作时 Pipeline 上各个 Stage 不是立即有 Instruction 而是需要经过一个 **Pipeline Fill** 阶段
逐步让各个 Stage 都进入有效 Instruction

>稳态阶段

Pipeline 已经填满 理想情况下 **每个 Cycle 完成一条 Instruction**

> 排空阶段

程序结束 或者遇到控制转移时 Pipeline 中剩余的 Instruction 仍然需要完成 **Pipeline Drain** 让已经进入Pipeline 的有效 Instruction 继续完成
