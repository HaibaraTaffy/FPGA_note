由于 Single-Cycle 的局限性 我们尝试将 Instruction 拆开 搭建 **Multi-Cycle CPU 多周期处理器**

于是一条指令可以拆分到多个周期完成
**Clock Period 只需要覆盖一个 Cycle 中最慢的 Hardware Operation**

## Intermediate State 中间状态
---
跨 Cycle 使用的数据必须保存 使用 Intermediate State

## Hardware Resource Reuse
---
即硬件资源复用 本质上是 **不同 Cycle 使用同一个 Hardware 完成不同事情**
即
```
Cycle 1
Hardware A → Operation 1
Cycle 2
Hardware A → Operation 2
Cycle 3
Hardware A → Operation 3
```

这是一种 **Trade-Off 权衡**

## 不要求每条 Instruction Cycle 数一样
---
不同 Instruction 可以执行不同数量的 Cycle

## Control 需要 State
---
