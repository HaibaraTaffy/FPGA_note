由于 每个 Cycle 需要的 Control Signal 完全不同 而当前执行到哪一步不能只从 Instruction Bits 中得出 所以 Control 本身需要保存执行进度 这就是引入 FSM 的直接原因

## FSM
---
Finite State Machine 有限状态机
通常包含三个主要部分
```
State Register
Next-State Logic
Output Logic
```
结构可以表示为
```
                   Input
                     │
                     ▼
Current State ──→ Next-State Logic
     │               │
     │               ▼
     │           Next State
     │               │
     │          Clock Edge
     │               │
     └──── State Register
                     │
                     ▼
                Current State
                     │
                     ▼
                Output Logic
                     │
                     ▼
              Control Signals
```

## State
---
表示 **控制器当前处于哪个执行步骤**
区分一些 State

| 保存内容           | 典型 Register          |
| -------------- | -------------------- |
| 当前 Instruction | Instruction Register |
| ALU 的中间结果      | ALU Result Register  |
| Memory 读出的数据   | Memory Data Register |
| 当前执行阶段         | FSM State Register   |
## Transition
---
State Transition 状态转移 
即**FSM 从 Current State 进入 Next State**
一般在 Clock Edge 时 
`FSM State Register <- Next State`

例如 Fetch Cycle：
```
Clock Edge 之前

Current State = FETCH
Instruction Memory 输出 Instruction
Control 允许 Instruction Register 写入
Next-State Logic 产生 DECODE
```
到达 Clock Edge：
```
Instruction Register ← Instruction
FSM State Register   ← DECODE
```
这两个更新在硬件语义上同时发生
Clock Edge 之后：
```
Current State = DECODE
Instruction Register 保存刚刚取出的 Instruction
```
接下来的组合逻辑才按照 `DECODE` 工作

## State 与 Cycle
---
```
Cycle = 时间单位
State = 控制器在这个时间段中的工作模式
```

同一个 State 可以持续多个 Cycle

## State 与 Control Signal
---
每个 State 对应一个明确的 Hardware Operation
例如：```FETCH```
要求：
```
Instruction Memory 使用 PC 取指
Instruction Register 允许写入
PC 产生下一取指地址
```
而：```ALU_WB```
要求：
```
Register File 允许写入
Write-Back Data 选择 ALU Result
Destination 选择 rd
```

**因此可以建立映射** :
```
Current State
↓
这一 State 要完成的 Operation
↓
需要启用的 Hardware Path
↓
Control Signals
```
**State 通过 Control Signal 选择 Datapath 中的数据路径**

但是也并不是 Control Signal 真的只由 Current State 决定
```
Current State 决定当前主要 Operation
Instruction Fields (指令字段) 与 Condition
进一步决定具体选择和 Transition
```

例如在 `DECODE`：
```
Opcode = LW
→ Next State = MEM_ADDR

Opcode = R-Type
→ Next State = EXEC_R
```
因此 Multi-Cycle Control 通常需要观察：
```
Current State
Instruction Opcode
funct3
ALU Condition
```
输出对应的控制信号

## 控制结果生效
---
Datapath中的组合逻辑可能一直在产生结果
只有对应的**写使能有效** 并且到达 Clock Edge
结果才真正被保存

所以 FSM 的关键作用不仅是选择 Data 从哪里走 还决定
**哪些 State Element 可以在这个 Clock Edge 更新**
典型写使能包括
```
PC Write Enable
Instruction Register Write Enable
Intermediate Register Write Enable
Register File Write Enable
Memory Write Enable
```

## 复位后
---
通常需要先从 `FETCH` State 开始
整体控制流程大致为
```
                    ┌─────────┐
                    │  FETCH  │
                    └────┬────┘
                         ▼
                    ┌─────────┐
                    │ DECODE  │
                    └────┬────┘
                         ▼
              根据 Instruction Type 分支
        ┌────────────┬────────────┬────────────┐
        ▼            ▼            ▼            ▼
      EXEC_R      MEM_ADDR      BRANCH        JUMP
        │         ┌───┴───┐        │            │
        ▼         ▼       ▼        │            │
      ALU_WB   MEM_READ MEM_WRITE   │            │
        │         │       │        │            │
        │         ▼       │        │            │
        │       MEM_WB    │        │            │
        └─────────┴───────┴────────┴────────────┘
                          │
                          ▼
                        FETCH
```

