## Multi-Cycle 的 State
---

| State Element      | 保存内容                 | 为什么保存                |
| ------------------ | -------------------- | -------------------- |
| IR                 | 当前 Instruction       | 后续多个 Cycle 继续 Decode |
| OldPC              | 当前 Instruction 原始 PC | PC 提前更新后仍能计算 Target  |
| A Register         | rs1 Data             | 后续 State 使用          |
| B Register         | rs2 Data             | 后续 State 使用          |
| ALUOut             | ALU Result 或 Address | 下一 State 使用          |
| MDR                | Memory Read Data     | Write-Back State 使用  |
| FSM State Register | 当前执行阶段               | Control 需要记住进度       |

## Multi-Cycle 的 State管理
---
Multi-Cycle 中同一条 `Instruction` 在不同 Cycle 需要不同控制
Multi-Cycle Control 依赖：
```
Current State 现状态
+
Instruction Fields 指令字段
+
Condition 条件
```

并且 要注意 `Write Enable` 的开关时刻 防止 寄存器在错误的时刻被修改
需要精确管理 `State Update Timing`

## Multi-Cycle 的 State Path
---

|Instruction|State Path|Cycle 数示例|
|---|---|---|
|R-Type|FETCH → DECODE → EXEC_R → ALU_WB|4|
|ADDI|FETCH → DECODE → EXEC_I → ALU_WB|4|
|LW|FETCH → DECODE → MEM_ADDR → MEM_READ → MEM_WB|5|
|SW|FETCH → DECODE → MEM_ADDR → MEM_WRITE|4|
|Branch|FETCH → DECODE → BRANCH|3|
|JAL|FETCH → DECODE → JAL|3|
|JALR|FETCH → DECODE → JALR|3|
这只是我们的 Microarchitecture 的设计 并非一定如此

## 时序 面积分析
---
- 多周期不一定快 速度比较应该看 `CPI × Clock Period` 而不是只看其一
- 多周期的 Clock Period 不会无限缩短 依旧受到时序限制 至少覆盖**最慢的State** 也没有消除Crtitical Path
- 多周期 通过复用 Hardware 可以减少一部分 Hardware 但是因为又同时增加了 其他的选择逻辑 于是**面积不一定更小**
- 多周期 可以让 Microarchitecture 的时间组织 更容易适配多周期或 同步 Hardware
- 多周期的 Microarchitecture Progress(微架构进程) 不应该被软件中断