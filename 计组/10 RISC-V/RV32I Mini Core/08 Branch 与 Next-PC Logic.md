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

---
**以下是扩展**

## 新增 BNE BLT BGE
---
- BNE - Branch if Not Equal - 不相等时分支
- BLT - Branch if Less Than - 小于时分支
- BGE - Branch if Greater Than or Equal - 大于等于时分支
加上原有的 
- BEQ - Branch if Equal - 相等时分支
**四条指令 Opcode 均为 `1100011` 且操作的组件相同 可以复用Main Decoder**

具体的 Branch 类型 由 `funct3` 判断

|指令|`funct3`|ALU 运算|跳转条件|
|---|---|---|---|
|BEQ|`000`|SUB|`zero`|
|BNE|`001`|SUB|`~zero`|
|BLT|`100`|SLT|`less_than`|
|BGE|`101`|SLT|`~less_than`|
其中 `less_than = alu_result[0]`

BLT BGE都是用 `SLT` 实现 所以可以复用 现有ALU
注 : SLT 小于则置位 A小于B 算出来为`32'b1` 即`less_than = alu_result[0]` 
正好符合跳转条件和指令

主要修改 `ALU_Decoder`

