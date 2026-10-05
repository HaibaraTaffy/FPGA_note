## 基本策略
---
先从 v2 复制一份成为 v3 基础

保留已验证过的基础模块
```
Instruction Memory 指令存储器
Register File 寄存器组
Immediate Generator 立即数生成器
Main Decoder 主译码器
ALU Decoder ALU 译码器
ALU 算术逻辑单元
Data Memory 数据存储器
```
围绕它们 重新组织 Datapath 和 Control

## 改造本质
---
将 单周期切分为多周期
```
IF
↓
IF/ID Register
↓
ID
↓
ID/EX Register
↓
EX
↓
EX/MEM Register
↓
MEM
↓
MEM/WB Register
↓
WB
```

核心变化
**原来属于一条 Instruction 的 Data 和 Control 必须跨多个 Clock Edge 保存 同时还要与其他 Instruction 的 Data 和 Control 分开**

## IF — Instruction Fetch — 取指
---
主要 State：```PC```
主要 Hardware：
```
Instruction Memory
PC + 4 Adder
Next-PC MUX
```
产生：
```
Instruction
Instruction PC
PC + 4
```
结果保存到：```IF/ID Register```

## ID — Instruction Decode — 指令译码
---
输入来自：```IF/ID Register```
主要 Hardware：
```
Main Decoder
ALU Decoder
Register File
Immediate Generator
```
产生：
```
rs1_data
rs2_data
Immediate
rs1
rs2
rd
funct3
ALU Control
后续 Stage 所需的 Control
```
结果保存到：```ID/EX Register```

## EX — Execute — 执行
---
输入来自：```ID/EX Register```
主要 Hardware：
```
Forwarding MUX
ALUSrc MUX
ALU
Branch Comparator
Branch / Jump Target Logic
```
完成：
```
Arithmetic / Logic Operation
Load / Store Address Calculation
Branch Compare
Branch Target Calculation
JAL Target Calculation
JALR Target Calculation
```
结果保存到：```EX/MEM Register```
Control Transfer 在 EX Stage 确认后：
```
Redirect PC
Flush 错误路径 Instruction
```

## MEM — Memory Access — 存储器访问
---
输入来自：```EX/MEM Register ```
主要 Hardware：```Data Memory```
完成：
```
LW
读取 Memory Data
SW
写入 Memory Data
ALU Instruction
传递 ALU Result
```
结果保存到：```MEM/WB Register```

## WB — Write Back — 写回
---
输入来自：```MEM/WB Register```
主要 Hardware：```Write-Back MUX```
写回来源：
```
00 → ALU Result
01 → Memory Read Data
10 → PC + 4
```
当有效时：
```RF[rd] ← Write-Back Result```

## 新建的文件
---
```
rtl/
├── rv32i_pipeline_core.v
├── pipeline_pc_reg.v
├── if_id_reg.v
├── id_ex_reg.v
├── ex_mem_reg.v
├── mem_wb_reg.v
├── forwarding_unit.v
└── hazard_unit.v
```

## IF/ID Register
---
保存
```
valid
instruction //在 ID 中译码
pc          //用于后续分支
pc_plus4    //用于分支写回
```

## ID/EX Register
---
保存
```
- Data:
  pc
  pc_plus4    //同上
  read_data1  //寄存器读出的操作数
  read_data2  //寄存器读出的操作数
  immediate   //立即数生成
- Register Address:
  rs1         //用于后续 Stall 或 flush
  rs2         //用于后续 Stall 或 flush
  rd          //用于后续 Stall 或 flush 以及 WB
- Instruction Information :
  funct3      //区分Instruction
  alu_control //控制ALU 用于 EX
- Control :
  reg_write   //用于 WB
  alu_src     //用于 EX
  mem_write   //用于 MEM
  result_src  //用于 WB
  branch      //用于 EX
  jump        //用于 EX
  jalr        //用于 EX
- State Information :
  valid
```
==几个Control信号需要回顾==

## EX/MEM Register 
---
保存
```
- Data:
  alu_result
  store_data 
  pc_plus4   //用于WB
- Register Address:
  rd         //用于 WB
- Control :
  reg_write  //用于 WB
  mem_write  //用于MEM
  result_src //用于 WB
- State Information :
  valid
```

其中 `alu_result` 可能表示
```
Arithmetic Result
Load Address
Store Address
```

尤其注意 `store_data` 与 `alu_result` 的区分
```
SW 的 Address 来自 alu_result
SW 的 Write Data 来自 rs2
```

## MEM/WB Register
---
保存
```
- Data:
  alu_result
  memory_read_data  
  pc_plus4   
- Register Address:
  rd         
- Control :
  reg_write
  result_src
- State Information :
  valid
```

`WB Stage` 根据 `result_src`
选择
```
ALU Result
Memory Read Data
PC + 4
```

## 引入 Valid Bit
---
即 有效位
定义：
```
valid = 1
Pipeline Register 中
保存一条有效 Instruction

valid = 0
Pipeline Register 中
是 Bubble 或无效内容
```

Valid Bit 用于处理：
```
Reset 后 Pipeline 尚未填满
Load-Use Stall 注入 Bubble
Branch Taken Flush
JAL / JALR Flush
```

最终有效写入条件 : 
```
Register File Write Enable =
(MEM/WB.valid) & (MEM/WB.reg_write) & ~rst

Data Memory Write Enable =
(EX/MEM.valid) & (EX/MEM.mem_write) & ~rst
```

## Pipeline Register 需要的基本操作
---
即
```
Reset
Clear
Enable
Normal Update
```
推荐优先级
```
Reset   //清空整个 Pipeline State
↓
Clear   //写入无效状态 用于 Bubble 或 Flush
↓
Enable  //正常接收上一级结果
↓
Hold    //保持当前内容 用于 Stall
```

## Testbench/Test Program
---
```
tb/
├── tb_pipeline_pc_reg.v
├── tb_if_id_reg.v
├── tb_id_ex_reg.v
├── tb_ex_mem_reg.v
├── tb_mem_wb_reg.v
├── tb_forwarding_unit.v
├── tb_hazard_unit.v
└── tb_rv32i_pipeline_core.v
```

```
programs/
├── pipeline_independent_test.hex
├── pipeline_forwarding_test.hex
├── pipeline_stall_test.hex
├── pipeline_flush_test.hex
└── pipeline_full_program_test.hex
```