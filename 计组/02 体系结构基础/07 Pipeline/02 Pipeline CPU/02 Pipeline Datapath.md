依旧采用 经典五级 Pipeline
**IF->ID->EX->MEM->WB**
暂时不处理
*Hazard Forwarding Stall Flush*
因此先假设 相邻 Instruction 之间无依赖

## Datapath 整体结构
---
```
┌──────────┐
│    PC    │
└────┬─────┘
     ├──────────────→ PC + 4
     ▼
Instruction Memory
     ▼
┌──────────────────┐
│  IF/ID Register  │
│  Instruction     │
│  PC              │
│  PC + 4          │
└────────┬─────────┘
         ▼
  Decode / RegFile / ImmGen
         ▼
┌──────────────────┐
│  ID/EX Register  │
│  rs1_data        │
│  rs2_data        │
│  Immediate       │
│  rd              │
│  PC / PC + 4     │
│  Control         │
└────────┬─────────┘
         ▼
    ALU / Compare / Target
         ▼
┌──────────────────┐
│ EX/MEM Register  │
│ ALU Result       │
│ Store Data       │
│ rd               │
│ PC + 4           │
│ Control          │
└────────┬─────────┘
         ▼
       Data Memory
         ▼
┌──────────────────┐
│ MEM/WB Register  │
│ ALU Result       │
│ Memory Data      │
│ PC + 4           │
│ rd               │
│ Control          │
└────────┬─────────┘
         ▼
   Write-Back MUX
         ▼
    Register File
```

## Pipeline Register
---
流水寄存器 **相邻流水线之间用于保存数据和控制信息的寄存器**
在我们的 五级 Pipeline 中 通常需要
```
PC Register
IF/ID Register
ID/EX Register
EX/MEM Register
MEM/WB Register
```
以上每个都是 **一组并行更新的Register**
可以理解成 **一组完整数据包**
例如 `ID/EX` 可能保存
```
rs1_data       32 bit
rs2_data       32 bit
Immediate      32 bit
PC             32 bit
PC + 4         32 bit
rd              5 bit
funct3          3 bit
ALU Control     若干 bit
Control         若干 bit
```
作用主要是 `保存阶段结果` 和 `隔离相邻 Stage`

## IF Stage
---
从 PC 中取出指令 保存指令与PC 并且完成PC顺序计算

> Clock Edge 更新

```
正常顺序执行时 :
PC <- PC + 4
同时更新 IF/ID Register
IF/ID.Instruction ← Instruction Memory[PC]
IF/ID.PC          ← PC
IF/ID.PCPlus4     ← PC + 4
```

对于同一条指令 PC 和 PC+4 需要跟着一起传递下去

## ID Stage
---
从 `IF/ID Register` 中取出 Instruction 然后送入不同的模块中进行**译指**
此时已经可以

- 从寄存器中读出 操作数1和2了
- 生成立即数
- 生成 Control 信号

> Clock Edge 更新

```
ID/EX.rs1_data  ← RF[rs1]
ID/EX.rs2_data  ← RF[rs2]
ID/EX.Immediate ← Immediate
ID/EX.PC        ← IF/ID.PC
ID/EX.PCPlus4   ← IF/ID.PCPlus4
//rd rs 为后续的 Forwarding 准备
ID/EX.rd        ← Instruction.rd
ID/EX.rs1       ← Instruction.rs1
ID/EX.rs2       ← Instruction.rs2
ID/EX.funct3    ← Instruction.funct3
//由于Decode会被后续指令复用
//所以需要保存当前译指产生的 Control
//使它跟着指令一起前进
ID/EX.Control   ← Decode 产生的 Control
```
但是并不是所有的指令 都必须保存完全相同的字段 
**而是后续 Stage 需要什么 Pipeline Register 就必须保存什么**

## Control 的划分
---
可以使用的阶段 划分Control

> EX Control

在 `EX Stage` 使用
```
ALUSrc     选择 ALU Operand
ALUControl 决定 ALU Operation
Branch     执行 Branch Compare
Jump       生成 Control-Flow Target
```

> MEM Control

在 `MEM Stage` 使用
```
MemWrite
MemRead
```

> WB Control

在 `WB Stage` 使用
```
RegWrite 是否写 RF
ResultSrc 选择写回 RF 的数据
```

## 如何使用产生的全部 Control Path
---
```
ID
产生全部 Control
↓
ID/EX
保存 EX + MEM + WB Control
↓
EX
使用 EX Control
↓
EX/MEM
保存 MEM + WB Control
↓
MEM
使用 MEM Control
↓
MEM/WB
保存 WB Control
↓
WB
使用 WB Control
```

## EX Stage
---
从 `ID/EX Register` 取得 指令 数据 和 控制信号

> ALU Operand A

```
ID/EX.rs1_data ─┐
                ├─→ Operand A MUX → ALU
ID/EX.PC ───────┘
```

> ALU Operand B

```
ID/EX.rs2_data ─┐
                ├─→ Operand B MUX → ALU
ID/EX.Immediate ┘
```

## 不同的 Instruction 对应的 EX 操作
---
### R-Type
```ALU Result ← rs1_data OP rs2_data```
### I-Type ALU
```ALU Result ← rs1_data OP Immediate```
### LW
```Memory Address ← rs1_data + Immediate```
### SW
```Memory Address ← rs1_data + Immediate```
### Branch
```
Compare Result ← rs1_data 与 rs2_data 比较
Branch Target ← PC + Immediate
```
### JAL
```Jump Target ← PC + Immediate```
### JALR
```Jump Target ← (rs1_data + Immediate) & ~1```

> Clock Edge 更新

```
//在Store中 rs2_data是需要保存的 ALU Result 是计算得出的 Address
EX/MEM.ALUResult ← ALU Result
EX/MEM.WriteData ← ID/EX.rs2_data
EX/MEM.rd        ← ID/EX.rd
EX/MEM.PCPlus4   ← ID/EX.PCPlus4
EX/MEM.MemWrite  ← ID/EX.MemWrite
EX/MEM.RegWrite  ← ID/EX.RegWrite
EX/MEM.ResultSrc ← ID/EX.ResultSrc
```

## MEM Stage
---
从 `EX/MEM Register` 中获得
```
ALU Result
Store Data
rd
PC + 4
MEM Control
WB Control
```
分类讨论

> LW

```
EX/MEM.ALUResult
↓
Data Memory Address
↓
Memory Read Data
```
其中 `ALU Result = Effective Address`

> SW

```
EX/MEM.WriteData			EX/MEM.ALUResult
↓							↓
Data Memory Write Data		Data Memory Address
```
其中 `MemWrite = 1` 在有效 Clock Edge : `Memory[Address] ← WriteData`

> Clock Edge 更新

```
MEM/WB.ALUResult ← EX/MEM.ALUResult
MEM/WB.ReadData  ← Data Memory Read Data
MEM/WB.PCPlus4   ← EX/MEM.PCPlus4
MEM/WB.rd        ← EX/MEM.rd
MEM/WB.RegWrite  ← EX/MEM.RegWrite
MEM/WB.ResultSrc ← EX/MEM.ResultSrc
```
不管用没用到 反正饱和打击就对了 总有指令会用到的

## WB Stage
---
从 `MEM/WB Register` 中获得数据
通过 `Write-Back MUX`
```
ResultSrc
├── ALU Result
├── Memory Read Data
└── PC + 4
```
得到 `Result`
在 `RegWrite = 1` 和 `Clock Edge` 时
得到 `RF[rd] <- Result`

## Instruction 不一定一直往下传
---
不需要的 Instruction Field 会被在使用完的阶段丢弃
只有需要保存的 Instruction Field 会被随着指令不断往下传

## 同步时序逻辑更新
---
同时更新 不会出现 **覆盖数据** 的情况
**原因是同步时序逻辑的基本语义**
```
所有 Register 在 Clock Edge
使用 Edge 前的输入值 更新
```

## Pipeline 的基本约束
---
**同一条 Instruction 的 Data Destination Control PC Information 必须始终保持对齐**

## 使用独立 Instruction Memory 和 Data Memory
---
使得基础 Pipeline Datapath 可以同时进行
`Instruction Fetch` 和 `Data Memory Access`

## Pipeline 中的 State
---
## Architectural State
```
PC Register
Register File
Data Memory 中程序可见内容
```

## Microarchitectural State
```
IF/ID Register
ID/EX Register
EX/MEM Register
MEM/WB Register
```

同时 也不能再用一个简单的 FSM State 表示 **整个 CPU 当前正在执行哪一步**
Pipeline的控制方式转变为
```
每条 Instruction 在 ID 生成自己的 Control
Control 随 Instruction 通过 Pipeline Register 前进
每个 Stage 使用属于当前 Instruction 的 Control
```