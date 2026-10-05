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

## 各个 Control 信号释义
---
>  Datapath

| 控制信号               | 作用                  | 生效阶段 | 控制硬件                | 二进制释义                                                                                             |
| ------------------ | ------------------- | ---- | ------------------- | ------------------------------------------------------------------------------------------------- |
| `imm_src[1:0]`     | 选择 Immediate Format | ID   | Immediate Generator | `00` I-Type `01` S-Type `10` B-Type `11` J-Type                                                   |
| `alu_op[1:0]`      | 选择 ALU 译码类别         | ID   | ALU Decoder         | `00` 直接 ADD `01` Branch Compare `10` 根据 `funct3/funct7` 译码 `11` Reserved                          |
| `alu_control[3:0]` | 选择具体 ALU Operation  | EX   | ALU                 | `0000` ADD `0001` SUB `0010` AND `0011` OR `0100` XOR `0101` SLL `0110` SLT `0111` SRL `1000` SRA |
| `alu_src`          | 选择 ALU Operand B    | EX   | ALU Operand B MUX   | `0` Forwarded Register Data `1` Immediate                                                         |
| `branch`           | 标记条件 Branch         | EX   | Branch Control      | `0` 非 Branch `1` 条件 Branch                                                                        |
| `jump`             | 标记 JAL              | EX   | Redirect Logic      | `0` 非 JAL `1` JAL                                                                                 |
| `jalr`             | 标记 JALR             | EX   | Redirect Logic      | `0` 非 JALR `1` JALR                                                                               |
| `mem_write`        | 控制 Data Memory 写入   | MEM  | Data Memory         | `0` 禁止写入 `1` 写入                                                                                   |
| `result_src[1:0]`  | 选择 Write-Back Data  | WB   | Write-Back MUX      | `00` ALU Result `01` Memory Read Data `10` PC + 4 `11` Reserved                                   |
| `reg_write`        | 控制 Register File 写入 | WB   | Register File       | `0` 禁止写入 `1` 写入                                                                                   |

> Branch Control

|控制信号|作用|生效阶段|控制硬件|二进制释义|
|---|---|---|---|---|
|`funct3[2:0]`|选择 Branch Condition|EX|Branch Condition Logic|`000` BEQ `001` BNE `100` BLT `101` BGE|
|`zero`|表示 ALU Result 是否为 0|EX|Branch Condition Logic|`0` 非零 `1` 为零|
|`less_than`|表示 Signed Operand A 是否小于 Operand B|EX|Branch Condition Logic|`0` 不小于 `1` 小于|
|`branch_condition`|表示条件 Branch 是否成立|EX|Redirect Logic|`0` 不成立 `1` 成立|
|`redirect`|请求修改 PC 并 Flush 年轻 Instruction|EX|PC MUX 与 Pipeline Control|`0` 顺序执行 `1` 跳转到 Target|
|`redirect_target[31:0]`|提供新的 PC|EX|PC Input MUX|32-bit Byte Address|
> Forwarding Control

| 控制信号             | 作用                  | 生效阶段 | 控制硬件                      | 二进制释义                                                               |
| ---------------- | ------------------- | ---- | ------------------------- | ------------------------------------------------------------------- |
| `forward_a[1:0]` | 选择最新 rs1 Data       | EX   | Operand A Forwarding MUX  | `00` ID/EX Data `01` MEM/WB Result `10` EX/MEM Result `11` Reserved |
| `forward_b[1:0]` | 选择最新 rs2 Data       | EX   | Operand B Forwarding MUX  | `00` ID/EX Data `01` MEM/WB Result `10` EX/MEM Result `11` Reserved |
| `store_forward`* | 为 Store Data 选择最新结果 | MEM  | Store Data Forwarding MUX | 具体编码在 Store Forwarding 实操时确定                                        |
> Pipeline State Control

| 控制信号            | 作用                                   | 生效阶段   | 控制硬件                     | 二进制释义                                    |
| --------------- | ------------------------------------ | ------ | ------------------------ | ---------------------------------------- |
| `rst`           | 清空 Processor Pipeline State          | 全部     | PC 与所有 Pipeline Register | `0` 正常 `1` Reset                         |
| `pc_enable`     | 控制 PC 是否更新                           | IF     | Pipeline PC Register     | `0` 保持 PC `1` 写入 Next PC                 |
| `if_id_enable`  | 控制 IF/ID 是否更新                        | IF/ID  | IF/ID Register           | `0` Hold `1` Update                      |
| `if_id_clear`   | Flush IF/ID                          | IF/ID  | IF/ID Register           | `0` 不清除 `1` 写入 Invalid/Bubble            |
| `id_ex_enable`  | 控制 ID/EX 是否更新                        | ID/EX  | ID/EX Register           | `0` Hold `1` Update                      |
| `id_ex_clear`   | Flush 或向 EX 注入 Bubble                | ID/EX  | ID/EX Register           | `0` 不清除 `1` 写入 Invalid/Bubble            |
| `ex_mem_enable` | 控制 EX/MEM 是否更新                       | EX/MEM | EX/MEM Register          | `0` Hold `1` Update                      |
| `ex_mem_clear`  | 清除 EX/MEM                            | EX/MEM | EX/MEM Register          | `0` 不清除 `1` 写入 Invalid/Bubble            |
| `mem_wb_enable` | 控制 MEM/WB 是否更新                       | MEM/WB | MEM/WB Register          | `0` Hold `1` Update                      |
| `mem_wb_clear`  | 清除 MEM/WB                            | MEM/WB | MEM/WB Register          | `0` 不清除 `1` 写入 Invalid/Bubble            |
| `valid`         | 标记 Pipeline Stage 是否包含有效 Instruction | 各级     | Pipeline Register 与副作用使能 | `0` Invalid/Bubble `1` Valid Instruction |
> Hazard Control

| 控制信号              | 作用                           | 生效阶段 | 控制硬件                   | 二进制释义                  |
| ----------------- | ---------------------------- | ---- | ---------------------- | ---------------------- |
| `uses_rs1`        | 表示 ID Instruction 是否真正读取 rs1 | ID   | Hazard Detection Unit  | `0` 不读取 `1` 读取         |
| `uses_rs2`        | 表示 ID Instruction 是否真正读取 rs2 | ID   | Hazard Detection Unit  | `0` 不读取 `1` 读取         |
| `load_use_hazard` | 表示 Consumer 必须等待 Load Result | ID   | Pipeline Control       | `0` 无 Hazard `1` Stall |
| `stall`           | 请求冻结前级并插入 Bubble             | ID   | PC IF/ID ID/EX Control | `0` 正常前进 `1` Stall     |
| `flush`           | 请求取消错误路径 Instruction         | EX   | IF/ID 与 ID/EX Control  | `0` 保留 `1` Flush       |