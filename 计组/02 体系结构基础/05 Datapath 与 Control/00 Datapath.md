**Datapath 本身只提供可能的路径
Control 决定选择哪条路径**

## 简化的 Datapath
---
```
                ┌─────────────┐
PC ────────────→│ Instruction │
                │   Memory    │
                └──────┬──────┘
                       │
                  Instruction
                       │
          ┌────────────┴────────────┐
          │                         │
          ▼                         ▼
    Register File                 ImmGen
       │      │                     │
       │      │                     │
       ▼      ▼                     │
    Read1   Read2 ───────┐          │
                         │          │
                         ▼          ▼
                            MUX
                             │
                             ▼
                            ALU
                             │
                             ▼
                         Data Memory
                             │
                             ▼
                            MUX
                             │
                             ▼
                       Register File
```

## Datapath 核心模块
---
先建立整体结构：

```
Datapath
│
├─ PC
│
├─ Instruction Memory
│
├─ Register File
│
├─ Immediate Generator
│
├─ ALU
│
├─ Data Memory
│
├─ Adders
│
└─ MUX
```

- Instruction Memory : 指令存储器 保存Instruction 不属于寄存器 属于存储器
- Data Memory : 数据存储器 保存程序运行过程中使用的数据

## 第一站 PC
---
由于 `PC中存的是 某个 Instruction Address`
于是 PC输出地址 给 **Instruction Memory**
```
PC
│ Instruction Address
▼
Instruction Memory
│ 
▼
32-bit Instruction
```
 例如 `PC = 0x1000`
 那么 PC 将地址`0x1000` 送给 Instruction Memory
 Instruction Memory 读出 `Address = 0x1000` 的Instruction 即`Instruction Memory[0x1000]`
 假设得到
 `add x5, x6, x7`

## 拆字段
---
Instruction 不是整个32bit 均送到同一个模块
不同的字段 会去 不同的地方
例如上面 :
```
Instruction
│
├─ rs1 ─→ Register File Read Address 1 寄存器端口的第一个读地址
├─ rs2 ─→ Register File Read Address 2寄存器端口的第二个读地址
├─ rd  ─→ Register File Write Address寄存器端口的写地址
│
└─ 其他字段 → Decode / Control
```

## 完整流程 ADD
---
`add x5, x6, x7`

> 取指

```
PC
↓
Instruction Memory
↓
add x5, x6, x7
```

> 读寄存器

Instruction 中：
```
rs1 = x6
rs2 = x7
```
送入 Register File：
```
x6 ─→ Read Address 1
x7 ─→ Read Address 2
```
得到：
```
Read Data 1 = RF[x6]
Read Data 2 = RF[x7]
```
假设：
```
RF[x6] = 20
RF[x7] = 7
```
那么输出：
```
20
7
```

> ALU计算

两个数据进入 ALU：
```
RF[x6] ─────┐
            │
            ▼
           ALU ─→ Result
            ▲
            │
RF[x7] ─────┘
```
执行：`20 + 7 = 27` 得到 `ALU Result = 27`

> 写回

Instruction 中：
```
rd = x5
```
所以：
```
Write Address = x5
Write Data    = 27
```
当 **Register Write Enable** 有效时：`RF[x5] <- 27`

> 整个 `add` 的主要 Datapath：

```
PC
↓
Instruction Memory
↓
Instruction
↓
rs1 / rs2
↓
Register File
↓
RF[x6] / RF[x7]
↓
ALU
↓
Result
↓
Register File
↓
x5
```

## 完整流程 ADDI 特点 ImmGen
---
现在看：
```
addi x5, x6, 10
```
Instruction 中提供：
```
rs1 = x6
rd  = x5
Immediate = 10
```
Register File 输出：
```
RF[x6]
```
与此同时 Instruction 送到：
```
Immediate Generator
```
得到：
```
32-bit Immediate = 10
```
于是：
```
RF[x6] ───────────┐
                  │
                  ▼
                 ALU
                  ▲
                  │
Immediate 10 ─────┘
```
然后：`ALU_Result = RF[x6] + 10`
最后：`RF[x5] ← ALUResult`

## MUX在Datapath 写入ALU 中的一个使用
---
由于 `ADD和ADDi` 的操作数不一样 但是不能直接把两根线同时接到 ALU
所以需要：
```
RF Read Data 2 ─────┐
                    │
                    ▼
                   MUX ─→ ALU Operand B
                    ▲
                    │
Immediate ──────────┘
```
这就是一个非常典型的 Datapath 共享


## 完整流程 lw 特点 Data Memory 写回
---
这一类型 还需要`访问 Data Memory`
```
lw x5, 8(x6)
```

> 第一步 读取 Base Register

由`rs1 = x6` 于是 RF输出 `RF[x6] = 0x1000(假设)`

> 第二步 生成 Immediate

由 `Immediate = 8` 经过 `ImmGen` 得到 `32-bit Immediate (假设) 8`

>第三步 计算 Effective Address

ALU 输入：
```
Operand A = RF[x6] = 0x1000
Operand B = 8
```
于是：`EA = 0x1000 + 8`
得到 `EA = 0x1008`
这里 ALU得到的是 `Memory Address` 而不是普通算数结果

> 第四步 访问 Data Memory

得到的 `ALU Result(0x1008)` 送给 Data Memory 的 Address
于是 `Data Memory` 输出 `Read Data`
假设：
`Memory[0x1008] = 123`
那么：
`Memory Read Data = 123`

>第五步 写回 Register File

由于 `rd = x5` 
所以对 RF 有
```
Write Address = x5
Write Data = 123
```
最终 `RF[x5] <- 123`

## Write Back MUX
---
写入RF的数据 可能来自 `ALU Result` 也可能来自 `Memory Read Data`
Register File 的 Write Data 需要选择
```
ALU Result ───────────┐
                     │
                     ▼
                    MUX ─→ Register File Write Data
                     ▲
                     │
Memory Read Data ─────┘
```

## 完整流程 SW 特点 Data Memory 不写回
---
`sw x5, 8(x6)`
```
                     ┌────────→ Data Memory Address
RF[x6] + Immediate → ALU
                     
RF[x5] ───────────────────────→ Data Memory Write Data
```

先从 `RF[x6]` 中取出基地址 和立即数 在ALU中计算出 EA 然后传给 `Data Memory Address` 端口 再从`RF[x5]`中取出数据 传给 `Data Memory Write Data` 端口

```
RF[x6]
  ↓
+ Immediate
  ↓
Address
  ↓
Memory
  ↑
RF[x5]
```

**不需要写回!**

## 完整流程 BEQ 特点 更新PC
---
`beq x5, x6, target`
语义:
```
如果 RF[x5] == RF[x6]
    跳到 target
否则
    继续 PC + 4
```

> 第一步 : 读取两个 Register

读取 `RF[x5]` 和 `RF[x6]` 然后比较 `RF[x5] == RF[x6]` 
得到一个条件结果 `Equal` 为1或0

>第二步 计算 Branch Target

概念上表示为 `Branch Target = PC + Branch Offset`
因此需要一个地址计算路径
```
PC ───────────────┐
                  ▼
                 Adder ─→ Branch Target
                  ▲
                  │
Branch Immediate ─┘
```
可能 ALU 正在计算 `Equal` 与此同时还需要计算 `BranchOffset` 
如果想在同一阶段 同时得到两个结果 需要两套加法能力

> 第三步 产生Next PC

```
PC + 4 ──────────┐
                 │
                 ▼
                MUX ─→ Next PC
                 ▲
                 │
Branch Target ───┘
```
采用 MUX 实现

>插入 : PC + 4的实现

并不是自动`+4` 硬件上必须真的存在某种加法逻辑
例如：
```
PC ──────┐
         ▼
       Adder ─→ PC + 4
         ▲
         │
         4
```

所以 其实需要多个Adder 是否共享 取决于微架构

## Datapath 中 三类概括
---
1. **State 状态** : 保存当前处理器状态 `PC Register File`
2. **Combinational Computation 组合计算** : 计算数据 `ALU Adder Comparator ImmGen`
3. **Selection 选择** : 选择路径 `Mux`

所以一个 CPU Datapath 可以大量概括成：
```
State
↓
MUX
↓
Combinational Logic
↓
MUX
↓
State
```

## Instruction 和 Datapath 关系
---
从微架构角度看 一条Instruction 理解成
**让CPU当前这一拍/这一段执行过程中 选择某些数据源 启用某些运算 允许某些状态更新**
Instruction 通过 Control 让 Datapath 呈现不同的数据流


