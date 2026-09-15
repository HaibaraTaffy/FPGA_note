**单周期处理器** 每条指令从开始执行到完成状态更新 只使用一个 Clock Cycle

这里的“一个周期”是指：
```
相邻两个有效 Clock Edge 之间
```
不是指整个程序只需要一个周期，也不是 IF、ID、EX、MEM、WB 各用一个周期。

## Single-Cycle 单周期
---
假设使用上升沿更新状态：
```
第 N 个上升沿
    ↓
当前 PC、RF 等状态确定
    ↓
一个周期内完成所有组合逻辑传播
    ↓
取指 → 译码 → 读 RF → ALU → 访问 Memory → 产生写回数据
    ↓
第 N+1 个上升沿
    ↓
PC、RF、Data Memory 等状态更新
```

**一条指令 在第N个Clock Edge 之后开始形成数据 在第 N+1 个Clock Edge 完成状态更新**

两条指令的关系可以表示为：
```
Clock Edge N
│
├─ Instruction 1 进行组合计算
│
Clock Edge N+1
│
├─ Instruction 1 在边沿 提交状态更新
├─ Instruction 2 在周期内 进行组合计算
│
Clock Edge N+2
│
├─ Instruction 2 在边沿 提交状态更新
├─ Instruction 3 在周期内 开始组合计算
```
每条指令占一个完整 Clock Cycle

如何实现?
在单周期 CPU 中：
```
一个 Clock Cycle
┌────────────────────────────────────────────┐
│ IF → ID → EX → MEM → WB                    │
└────────────────────────────────────────────┘
```
阶段之间没有用 Register 隔开。
数据只是连续经过组合逻辑：
```
PC
↓
Instruction Memory
↓
Decode / Register File
↓
ALU
↓
Data Memory
↓
Write-Back MUX
↓
Register File Write Data
```
直到下一个 Clock Edge 到来 最终状态才更新

## 一个周期中的 State Data Control
---
### 1. State
单周期 CPU 中需要在 Clock Edge 更新的主要状态包括：
```
PC
Register File
Data Memory
```
Instruction Memory 通常在运行过程中只读 其内容由程序加载阶段提前准备好
### 2. Data
一个周期内 数据从当前状态出发 经过组合逻辑形成：
```
Instruction
Register Operand Values
Immediate
ALU Result
Memory Read Data
Write-Back Data
Next PC
```
### 3. Control
Instruction 被译码后产生：
```
RegWrite
ALUSrc
ALUControl
MemWrite
ResultSrc
Branch
PCSrc
```
这些信号在同一个周期内配置整条 Datapath 

## ADD例子
---
`add x5, x6, x7`
假设在第 N 个 Clock Edge 后：
```
PC = 0x1000
RF[x6] = 20
RF[x7] = 7
```
**周期内**发生 (即组合逻辑计算)：
```
PC
→ Instruction Memory
→ add x5, x6, x7
→ 读取 RF[x6] 和 RF[x7]
→ ALU 计算 20 + 7
→ ALU Result = 27
→ Write-Back MUX 选择 ALU Result
→ RF Write Data = 27
```
同时：
`Next PC = PC + 4 = 0x1004`
控制信号为：
```
RegWrite   = 1
ALUSrc     = 0
ALUControl = ADD
MemWrite   = 0
ResultSrc  = ALU
PCSrc      = PC + 4
```
到第 N+1 个 Clock Edge：
```
RF[x5] ← 27
PC     ← 0x1004
```
`RF` 和 `PC` 可以在同一个 Clock Edge 更新 因为它们是不同的状态存储结构 不会出现读写冲突

## LW例子
---
`lw x5, 8(x6)`
假设：
```
PC = 0x1000
RF[x6] = 0x2000
Memory[0x2008] = 100
```
一个周期内的数据路径为：
```
PC
→ Instruction Memory
→ Instruction Decode
→ Register File 读取 RF[x6]
→ ImmGen 生成 Immediate 8
→ ALU 计算 0x2000 + 8
→ Data Memory 读取 Memory[0x2008]
→ Write-Back MUX 选择 Memory Read Data
→ RF Write Data = 100
```
同时：
`Next PC = 0x1004`
在下一个 Clock Edge：
```
RF[x5] ← 100
PC     ← 0x1004
```
注意 在 Clock Edge 到来之前 `RF Write Data`、`Write Address`、`RegWrite` 等信号都必须稳定并满足时序要求

## Critical Path : LW
---
- **Propagation Delay 传播延迟** : 输入发生变化后 信号经过硬件并使输出稳定所需要的时间
- **Critical Path 关键路径** : 整个同步电路中 传播延迟最长 限制最高 Clock Frequency 的**组合逻辑路径**

由于 `lw` 依次经过
```
Instruction Memory
→ Register File
→ ALU
→ Data Memory
→ Write-Back MUX
```
几乎是 单周期CPU 最长的组合逻辑路径 于是要求 **时钟周期必须足够长 使这条路径在下一个 Clock Edge 前稳定**

## Single-Cycle CPU的局限
---
**由于 单时钟CPU只能使用同一时钟 所以 Clock Period 必须迁就 Critical Path**
就会出现 "快等慢" 即**所有指令都使用由最慢指令决定的 Clock Period** 较简单的指令无法提前结束

## CPI
---
Cycles Per Instruction 即 **平均每条指令所需的时钟周期数**
理想的 单周期 CPU `CPI = 1`
`CPU执行时间 = Instruction Count 指令个数 * CPI * Clock Period`
可见 **CPI和Clcok Frequency 共同决定了 CPU的执行时间**
CPI较小 但不一定执行时间小 因为单周期CPU的Clock Period 很长

## Harvard Architecture
---
即 哈佛结构 指令与数据存储通路分开的组织方式 
在我们的单周期CPU中 表现为
```
Instruction Memory
Data Memory
```
两个独立的访问通路
由于执行 `lw` 或 `sw` 时，一个周期内可能同时需要：
1. 从 Instruction Memory 读取当前指令；
2. 访问 Data Memory。
如果指令和数据共用一个单端口存储器，就不能在同一时刻完成两次独立访问

## 单周期CPU的特点
---
- Clock Period 较大
- 取指和数据访问同时进行
- 复制部分计算硬件以在一周期内完成所有工作

优点：
- 控制过程相对直接；
- 所有指令都是一个周期；
- 不需要处理 Pipeline Hazard；
- 适合学习完整 Datapath；
- 适合验证指令语义与控制信号。
缺点：
- Clock Period 由最慢指令决定；
- 简单指令也必须等待完整周期；
- 关键路径可能很长；
- 为保证并行计算，可能需要更多独立硬件；
- 对存储器读取方式有严格要求；
- 不适合追求高 Clock Frequency。

## FPGA部署注意
---
FPGA中的 Bram 通常采用同步读取 会产生一些时序问题 需要注意
**FPGA 部署模型**：根据 BRAM 的同步读取特性调整时间组织，可能需要加入取指状态或流水级。