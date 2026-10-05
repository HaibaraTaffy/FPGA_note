即 流水线性能
性能分析的意义
**找到真正限制系统的部分 再决定该优化什么**

## 对 CPU Time
---
Pipeline 可能可以 `缩短 Clock`
但是 Hazard 又可能增加 `CPI`
最终需要比较 `CPI * Clock Period`

## Latency 与 Throughput
---
Latency 延迟 对于五级 Pipeline 
若每个 Stage 为一个 Cycle
那么 `单条 Instruction Latency ≈ 5 Cycles`

Throughput 吞吐率
Pipeline 填满后 理想情况下
`每一个Cycle 完成一条 Instruction`
但是最终衡量吞吐率 还是要分析一下 `每周期的操作数`

**Pipeline 的主要收益是 提高 Throughput**

## Pipeline 的 CPI
---
对于五级流水线 N条指令
`Total Cycles = N + 4`
`平均 CPI = (N + 4)/N`
当 N 很大时 `CPI -> 1`

但是理想 CPI 并不是永远等于1
因为存在
`Pipeline Fill 流水线填充` 和 `Pipeline Drain 流水线排空`

实际上 会由于
```
Load-Use Stall
Branch Flush
Structural Stall
Memory Wait
```
导致产生额外 Cycle

## Forwarding 贡献
---
主要性能收益 `降低 CPI`
但是可能增加 `Hardware` 进而增加 `Clock Period`
最终还是需要做一个 Trade off

## IPC
---
 Instruction Per Cycle 每周期完成的指令数
 `IPC = 1/CPI`
 可以表示 Instruction Throughput

## Stage Balance
---
由于 **Clock Period 由最慢 Stage 决定**
所以需要考虑合理分配各个 Stage 的时长
使得最慢 Stage 不致于 拖慢整个Pipeline

## Register Overhead
---
寄存器开销 关于时序收敛与稳定 所以 Pipeline Stage 不能无限增加

## Memory Stall
---
之前我们假设 
```
Instruction Memory 每个 Cycle 返回 Instruction
Data Memory 在一个 MEM Stage 内完成
```
但是真实的 `Memory System` 可能需要等待
一次 `Cache Miss` 可能需要等待几十个 Cycles
此时 Pipeline 可能必须
```
冻结相关 State
等待 Memory Response
阻止 Instruction 丢失或重复执行
```

平均 `Memory Penalty` 近似为
`Memory Instruction Frequency * Miss Rate * Miss Penalty`

例如：
```
Load / Store 比例 = 30%
Miss Rate = 5%
Miss Penalty = 20 Cycles
```
得到：
```
0.30 × 0.05 × 20
=
0.30 Cycle / Instruction
```

>注 Cache Hit 缓存命中 Cache Miss 缓存未命中
>CPU要访问的数据在Cache中有缓存 称为“命中” (Hit)，反之则称为“缺失” (Miss)

## AMAT
---
即 Average Memory Access Time 平均存储访问时间
`AMAT = Hit Time + Miss Rate * Miss Penalty`

注意 **Pipeline 每个 Cycle 可以接收新 Instruction
不代表 Memory 每个 Cycle 都能持续提供 Instruction 和 Data**

## Instruction Mix
---
指令组合 程序中不同类型指令所占比例
由于指令不同 所以产生的额外 Cycle 也会不同
Processor 性能必须结合 `Workload` 进行判断

## Amdahl's Law
---
**阿姆达尔定律**
如果系统中某部分占原执行时间比例：```P```
该部分被加速：```S 倍```
整体 Speedup：
```
Overall Speedup = 1 ÷ ((1 - P) + P / S)
```

## Retire
---
即 Instruction Retirement 指令退役
表示一条 Instruction 已经完成  
其 Architectural Effect 被正式接受
**并不是每一次 Fetch 都记为完成一条 Instruction
需要加以区分**

## Performance Counter
---
性能计数器
可以统计
```
Total Cycles 总周期
Retired Instructions 退役指令
Load-Use Stall Cycles 存取停顿周期
Branch Count 分支数
Branch Taken Count 分支采用数
Branch Misprediction Count 分支预测失误数
Instruction Memory Wait Cycles 指令存取等待周期
Data Memory Wait Cycles 数据存取等待周期
```
然后计算
```
CPI
IPC
Branch Misprediction Rate 分支预测失误率
Memory Stall Ratio 内存阻塞比率
```
Performance Counter 的价值不是改变执行结果
而是让 Microarchitecture 的瓶颈可以被观察和量化

## Pipeline Utilization
---
利用率 即 **Hardware 处于有效工作状态的比例**

## Arithmetic Intensity
---
算术强度 即 **每搬运一定数量数据所执行的计算量**

```
Arithmetic Intensity
= Operations ÷ Memory Bytes Transferred
= 操作 ÷ 内存传输字节
```
