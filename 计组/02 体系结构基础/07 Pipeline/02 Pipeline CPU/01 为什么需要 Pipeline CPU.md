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