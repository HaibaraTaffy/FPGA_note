## 基础 Multi-Cycle Datapath
---
```
PC    指令计数器 存放当前指令地址
OldPC 旧指令计数器 存放当前Instruction 原来的PC
Instruction Register IR 指令寄存器 存放当前指令
Register File 寄存器组
A Register A 寄存器 保存 rs1 的数据
B Register B 你存器 保存 rs2 的数据
Immediate Generator 立即数生成器
ALU 算术逻辑单元
ALUOut Register ALU结果寄存器 保存ALU的跨周期结果
Data Memory 数据存储器
Memory Data Register MDR 存储器数据寄存器
Write-Back MUX 写回多选
FSM Controller 状态机控制器
```

## OldPC的存在需要
---
假设 Fetch State 同时执行：
```
IR ← Instruction Memory[PC]
PC ← PC + 4
```
Clock Edge 之后：
```PC 已经指向下一条 Instruction```
但是当前 Instruction 的 Branch 或 JAL Target 仍需要：
```当前 Instruction 原来的 PC```
所以 Fetch 时还要保存：
```OldPC ← PC```
于是：
```
OldPC  保存当前 Instruction 的地址
PC 提前指向顺序执行的下一条 Instruction
```

## FETCH State
---
> Data来源

`PC Register`

> Hardware 路径

路径1
```
PC
↓
Instruction Memory
↓
Instruction
```

路径2
```
PC
↓
ALU

4
↓
ALU
```
在ALU中执行 `PC + 4`

>Clock Edge 更新

```
IR    ← Instruction Memory[PC] (组合逻辑的锁存)
OldPC ← PC
PC    ← PC + 4 (组合逻辑的锁存)

FSM State ← DECODE
```

## DECODE State
---
> Data 来源

`IR` 提供 *Instruction*
`RF` 根据 *rs1 rs2* 读出 **rs1_data rs2_data**
`ImmGen` 则根据 *Instruction* 产生 **Immediate**

> Datapath

```

IR.rs1     		IR.rs2          
↓				↓
Register File	Register File
↓				↓
rs1_data		rs2_data
↓				↓
A Register		B Register
```
同时 复用ALU 预先计算 控制流 Target
```
OldPC + Immediate
↓
ALU
↓
ALUOut Register
```

> Clock Edge 更新

Datapath 保存 Operand 和可能的 Target
```
A      ← RF[rs1]
B      ← RF[rs2]
ALUOut ← OldPC + Immediate
```

Control 识别 Instruction Type 并选择后续路径
```
R-Type                → EXEC_R
ADDI 或其他 I-Type ALU → EXEC_I
LW / SW               → MEM_ADDR
Branch                → BRANCH
JAL                   → JAL
JALR                  → JALR
```

**对于不是 Branch 或 JAL 的 Instruction**
`OldPC + Immediate` 可能没有用途 但不产生错误
后续 State 不一定使用它


## R-Type Instruction Datapath
---
以 `ADD x3 x1 x2` 为例

>完整 State Path：

```
FETCH
↓
DECODE
↓
EXEC_R
↓
ALU_WB
↓
FETCH
```

## EXEC_R State
---
> Data 来源

几乎后续所有指令都需要经过 `FETCH` 和 `DECODE` 
此时的 Operand已经存入 `A Register 和 B Register` 中 
为默认条件 于是
```
A Register → RF[x1] 从寄存器A中读出 原x1中的数据
B Register → RF[x2]
```

> Hardware 路径

```
A Register              B Register               
↓RF[x1]					↓RF[x2] 
ALU Operand A			ALU Operand B
将 A B 寄存器中的数据 驱动到 Operand A和B线上
```
ALU Operation 由：`funct3 和 funct7` 决定
对于 `ADD` 指令 
```
A + B
↓
ALU Result
```

> Clock Edge 更新

```
ALUOut ← A + B
FSM State ← ALU_WB
```
此时 `x3` 未改变 这一 State 只保存运算结果

## ALU_WB State
---
> Datapath

```
ALUOut
↓
Write-Back MUX
↓
Register File Write Data
```

IR 提供 `rd = x3`

> Clock Edge 更新

```
RF[x3] ← ALUOut
FSM State ← FETCH
```
此时 `x3` 改变 获得新的值
**完成ADD Instruction**


## I-Type ALU Instruction 数据流
---
以 `ADDI x3 x1 8` 为例

>完整 State Path：

```
FETCH
↓
DECODE
↓
EXEC_I
↓
ALU_WB
↓
FETCH
```

## EXEC_I State
---
> Datapath

```
A Register			Immediate
↓					↓
ALU Operand A		ALU Operand B
```
ALU 执行 `A + Immediate`

> Clock Edge 更新

`ALUOut ← A + Immediate`

# ALU_WB State
---
> Datapath

```
ALUOut
↓
Write-Back MUX
↓
RF[rd]
```

> Clock Edge 更新

`RF[rd] <- ALUOut`
**完成 ADDI Instruction**


## LW Datapath
---
以 `LW x5 12(x1)` 为例

先分析语义 :
```
Address ← RF[x1] + 12
RF[x5]  ← Memory[Address]
```

> 完整State Path :

```
FETCH
↓
DECODE
↓
MEM_ADDR
↓
MEM_READ
↓
MEM_WB
↓
FETCH
```

## MEM_ADDR State
---
> Data 来源

```
A Register = RF[x1]
Immediate  = 12
```

> Datapath

```
A				Immediate
↓				↓
ALU Operand A   ALU Operand B

ALU
↓
A + Immediate
```

> Clock Edge 更新

`ALUOut ← A + Immediate`
此时 `ALUOut` 中保存的是 **Memory Address**

## MEM_READ State
---
> Datapath

```
Data Memory Read Data	ALUOut
↓						↓
MDR						Data Memory Address
```

> Clock Edge 更新

`MDR ← Data Memory[ALUOut]`
此时的 `RF[x5]` 未改变 
*Memory Data* 暂时保存在 `MDR` 中

## MEM_WB State
---
> Datapath

```
MDR
↓
Write-Back MUX
↓
Register File Write Data
```
此时 IR 提供 `rd = x5`

> Clock Edge 更新

`RF[x5] ← MDR`
**完成 LW Instruction**


## SW Datapath
---
以 `SW x5 12(x1)` 为例

先分析语义 :
```
Address ← RF[x1] + 12
Memory[Address] ← RF[x5]
```

> 完整 State

```
FETCH
↓
DECODE
↓
MEM_ADDR
↓
MEM_WRITE
↓
FETCH
```

## MEM_ADDR State
---
> Data 来源

```
A Register = RF[x1]
Immediate  = 12
```

> Datapath

```
A				Immediate
↓				↓
ALU Operand A   ALU Operand B

ALU
↓
A + Immediate
```

> Clock Edge 更新

`ALUOut ← A + Immediate`
此时 `ALUOut` 中保存的是 **Memory Address**

## MEM_WRITE State
---
> Datapath

```
ALUOut			     B Register
↓			          ↓
Data Memory Address	Data Memory Write Data
```

> Clock Edge 更新

`Data Memory[ALUOut] <- B`

## Branch Datapath
---
以 `BEQ x1 x2 target` 为例

>完整 State path

```
FETCH
↓
DECODE
↓
BRANCH
↓
FETCH
```

注意 在 Decode State 已经完成
```
A ← RF[x1]
B ← RF[x2]
ALUOut ← OldPC + Branch Immediate
```
所有需要的数据已经准备好

## BRANCH State
---
> Datapath

比较 路径 :
```
A		B
↓		↓
ALU		ALU

ALU : A - B -> zero
```
Target 路径 :
```
ALUOut
↓
PC Input MUX
```
Control 路径 决定 PC 是否写入
```
funct3
+
ALU Condition
```
注意 `FETCH 时 PC = OldPC + 4`

> Clock Edge 更新

```
Branch Taken : 
PC Write Enable = 1
PC <- ALUOut
Branch Not Taken : PC保持原值
```

## JAL Datapath
---
先分析语义
```
RF[rd] ← OldPC + 4
PC ← OldPC + Immediate
```

注意到
```
FETCH 后 PC = OldPC + 4
DECODE 后 ALUOut = OldPC + Immediate
```
正是我们所需要的 并且由于直接跳转 所以直接可以使用

## JAL State
---
> Datapath

```
PC							ALUOut
↓							↓
Write-Back MUX				PC Input MUX
↓							↓
RF[rd]						PC
```

> Clock Edge 更新

```
RF[rd] ← PC
PC     ← ALUOut
```

## JALR Datapath
---
先分析语义
```
RF[rd] ← OldPC + 4
PC ← (RF[rs1] + Immediate) & ~1
```
注意到
```
在 FETCH 后
PC = OldPC + 4
在 DECODE 后
Register A = RF[rs1]
```
已经有一部分 Data准备好

## JALR State
---
```
path1 			path 2
A				PC
+				↓
Immediate		Write-Back MUX
↓				↓
ALU				RF[rd]
↓				
清除 bit[0]				
↓				
PC Input				
```

> Clock Edge 更新

```
RF[rd] ← PC
PC ← {ALU Result[31:1], 1'b0}
```
由于同步更新 不涉及跨时钟保存 这里的 ALU Result 可以直接写入 PC

