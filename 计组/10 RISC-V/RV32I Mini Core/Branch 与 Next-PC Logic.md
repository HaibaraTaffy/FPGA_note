## 模块职责
---
Next-PC Logic负责计算：
```
PC + 4
Branch Target
Branch是否成立
最终Next PC
```
核心关系：
```
pc_plus4      = current_pc + 4
branch_target = current_pc + immediate
pc_src        = branch & zero
```
最终选择：
```
pc_src = 0 → next_pc = pc_plus4
pc_src = 1 → next_pc = branch_target
```

## branch、zero和pc_src
---
`branch`
表示当前 Instruction属于条件Branch。
```zero```
表示 ALU Result为0。
对于第一版只支持的 `beq`：
`PCSrc=branch AND zero`
只有两个条件同时成立，才选择 Branch Target。

## State与组合逻辑
---
Next-PC Logic只负责计算 Next State，不保存状态。
```
Next-PC Logic → 组合逻辑
PC Register   → 状态存储
```
完整关系：
```
Current PC
    ↓
Next-PC Logic
    ↓
next_pc
    ↓
PC Register
    ↓
New Current PC
```
核心结论：
**Next-PC Logic计算Next State，PC Register保存State**