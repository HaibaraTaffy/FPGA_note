![[Pasted image 20260916210032.png]]
第一颗CPU成功跑起来了!!!

## 功能
---
```
例化各个子模块
拆分Instruction字段
连接 Data Signal
连接 Control Signal
实现 ALUSrc Mux
实现 Write-Back MUX
```

## 架构
---
```
PC Register
    ↓
Instruction Memory
    ↓
Instruction
    ├─ Opcode ─────────→ Main Decoder
    ├─ funct3/funct7 ─→ ALU Decoder
    ├─ rs1/rs2/rd ────→ Register File
    └─ Immediate Bits → ImmGen

Register File
    ↓
ALUSrc MUX
    ↓
ALU
    ├─→ Data Memory
    ├─→ Write-Back MUX
    └─→ Zero

Data Memory
    ↓
Write-Back MUX
    ↓
Register File

Branch + Zero
    ↓
Next-PC Logic
    ↓
PC Register
```

## 支持的指令
---
```
add sub and or xor addi lw sw beq
```
## 字段拆分
---
```Verilog
assign opcode      = instruction[6:0];
assign rd          = instruction[11:7];
assign funct3      = instruction[14:12];
assign rs1         = instruction[19:15];
assign rs2         = instruction[24:20];
assign funct7_bit5 = instruction[30];
```

**虽然顶层始终截取这些字段 但并非每条指令都会使用全部字段**
**所以会导致 有些指令会导致无关的信号也变化 但是不影响正常功能**


## ALUSrc MUX
---
ALU的 Operand A固定来自：```RF[rs1]```
Operand B可能来自：
```
RF[rs2]
Immediate
```
选择逻辑：
```Verilog
assign alu_operand_b =
    alu_src ? immediate : read_data2;
```
对应：
```
alu_src = 0 → RF[rs2]
alu_src = 1 → Immediate
```

## Write-Back MUX
---
写回 Register File的数据可能来自：
```
ALU Result
Memory Read Data
```
选择逻辑：
```Verilog
assign write_back_data =
    result_src ? memory_read_data : alu_result;
```
对应：
```
result_src = 0 → ALU Result
result_src = 1 → Memory Read Data
```

## 不同指令的流程
---
## `add`
```
RF[rs1]
RF[rs2]
   ↓
ALU ADD
   ↓
Write-Back MUX选择ALU Result
   ↓
RF[rd]
```

## `addi`
```
RF[rs1]
Immediate
   ↓
ALU ADD
   ↓
RF[rd]
```

## `lw`
```
RF[rs1] + Immediate
   ↓
Effective Address
   ↓
Data Memory
   ↓
Write-Back MUX选择Memory Read Data
   ↓
RF[rd]
```

## `sw`
```
RF[rs1] + Immediate → Data Memory Address
RF[rs2]             → Data Memory Write Data
```

## `beq`
```
RF[rs1] - RF[rs2]
        ↓
       Zero

Branch & Zero
        ↓
      PCSrc
        ↓
Next-PC MUX
```

## 状态更新时间
---
在**一个时钟周期内 组合逻辑完成**
```
取指
→ 指令字段拆分
→ 控制信号生成
→ 读取 Register File
→ 生成 Immediate
→ ALU 运算
→ 访问 Data Memory
→ 选择 Write-Back Data
→ 计算 Next PC
```

在**时钟上升沿**更新状态：
```
PC ← next_pc
```
如果允许写寄存器：
```
RF[rd] ← write_back_data
```
如果允许写 Data Memory：
```
Memory[address] ← write_data
```
因此可以将单周期CPU理解为:
**一个周期内完成组合计算 在周期末的上沿保存计算结果**

## NOP指令
---
Instruction Memory 未加载程序的区域被初始化为：```00000013```
对应：
```addi x0, x0, 0```
它会执行：
```0 + 0 → x0```
虽然 `reg_write = 1` 但 Register File 禁止写入 `x0` 因此不会改变 CPU 状态
该指令作为 NOP（No Operation 无操作指令）使用

## 复位期间 禁止写入
---
```Verilog
wire reg_write_enable;
wire mem_write_enable;

assign reg_write_enable = reg_write & ~rst;
assign mem_write_enable = mem_write & ~rst;

.write_enable(reg_write_enable)
.write_enable(mem_write_enable)
```
保证了 `RF` 和`Data Memory` 禁止写入

## 本节结论
---
单周期 CPU 会同时产生多个候选计算结果。
控制信号和 MUX 决定当前指令真正使用的数据路径。
没有被选择的组合信号可能出现看似奇怪的值，但只要它：
```
没有被 MUX 选中
并且没有触发状态写入
```
就不会影响 CPU 的执行结果。

本节已经完成：
```
PC
Instruction Memory
Decoder
Register File
Immediate Generator
ALU
Data Memory
Next-PC Logic
```
之间的顶层连接，并成功执行了第一段 RV32I 程序。