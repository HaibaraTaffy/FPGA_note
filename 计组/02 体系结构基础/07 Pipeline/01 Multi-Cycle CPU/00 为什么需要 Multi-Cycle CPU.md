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
因为 Instruction 被拆分成不同的 State 在不同的 State 中 需要不同的 Control Signal
所以现在的 Control 必须知道 
`Instruction Type + Current Execution Stage`
需要知道
```
上一 Cycle 在哪
这一 Cycle 应该做什么
下一 Cycle 去哪
```

引入 `FSM 有限状态机`

## 两个层次的 State
---
- Architectural State 架构状态 : `PC RF Memory中程序可见数据`
- Microarchitectural State 微架构状态 : `Control 当前执行到哪一步等`

## 打破指令与周期严格绑定
---
```
Instruction
        ↓
拆成多个 Operation
        ↓
分布到多个 Cycle
        ↓
Cycle 之间保存 Intermediate State
        ↓
Hardware 可以跨 Cycle 复用
        ↓
Control 根据当前执行步骤变化
```

## Multi-Cycle Datapath 的基本结构
---
```
                       ┌───────────────┐
                       │ Instruction   │
                       │    Memory     │
                       └───────┬───────┘
                               ▼
                       ┌───────────────┐
                       │ Instruction   │
                       │   Register    │
                       └───────┬───────┘
                               ▼
                           Decode
                    ┌──────────┴──────────┐
                    ▼                     ▼
              Register File           Immediate
                    └──────────┬──────────┘
                               ▼
                         ┌─────────┐
                         │   MUX   │
                         └────┬────┘
                              ▼
                           ┌─────┐
                           │ ALU │
                           └──┬──┘
                ┌─────────────┼─────────────┐
                ▼             ▼             ▼
           Intermediate    Data Memory    PC Logic
             Register
                │
                ▼
             MUX
                │
                ▼
          Register File
```

**需要 Instruction Register 来保证当前正在执行的 Instruction 不会因为 PC 改变而丢失**

```
PC
 ↓
取指
Clock Edge
 ↓
Instruction 被锁存
后面的多个 Cycle
 ↓
一直使用这个 Instruction
```
扩展 以下都可能需要成为 **Intermediate State**
```
Instruction
Address
Memory Data
ALU Result
```

## 对照 Single-Cycle
---

|                    | Single-Cycle       | Multi-Cycle                 |
| ------------------ | ------------------ | --------------------------- |
| Instruction 完成     | 1 Cycle            | 多个 Cycle                    |
| Clock Period       | 受最长 Instruction 限制 | 只需要覆盖单个 Cycle 的关键路径         |
| Hardware           | 较多并行资源             | 更强调资源复用                     |
| MUX                | 有                  | 更多                          |
| Intermediate State | 较少                 | 明显增加                        |
| Control            | 主要由 Instruction 决定 | Instruction + Current State |
| FSM                | 不需要作为核心控制机制        | 成为核心控制机制                    |
| Datapath           | 一条完整执行路径           | 多个 Cycle 共享 Datapath        |

