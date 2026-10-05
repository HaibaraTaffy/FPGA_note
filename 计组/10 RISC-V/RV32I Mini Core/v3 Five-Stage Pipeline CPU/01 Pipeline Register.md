本课完成
```
pipeline_pc_reg.v
if_id_reg.v
id_ex_reg.v
ex_mem_reg.v
mem_wb_reg.v
以及对应的 5 个 Testbench
```

## 统一规则
---
上升沿 `posedge` 更新 
控制优先级 `Reset -> Clear -> Enable -> Hold`
含义
```
rst = 1 → State 清零
clear = 1 → 当前内容变成 Bubble
enable = 1 → 保存输入
enable = 0 → 保持原值 即Hold
```

