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
结构冒险 多个 **流水级** 在同一 Cycle 争用同一个硬件资源

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

