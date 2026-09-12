即 Arithmetic Logic Unit 算术逻辑单元

## 简介
---
是CPU中 负责完成一类基础数据运算的 组合逻辑单元
最典型的功能 包括 :

- Arithmetic Operation —— 算术运算
- Logic Operation —— 逻辑运算
- Compare —— 比较
- 有些实现还会包含 Shift —— 移位

通常可能由ALU完成的指令
```
ADD
SUB
AND
OR
XOR
SLT
```

将ALU理解为 **一个可以根据控制信号选择不同运算功能的组合运算单元**

**ALU本身属于 Combinational Logic 并不是Register**
所以 ALU本身通常不负责长期保存结果 只负责**计算**

## 基本接口 
---
一个典型 ALU 可以抽象成：
```
                ┌──────────────┐
Operand A ─────→│              │
Operand B ─────→│     ALU      ├────→ Result
ALU Control ───→│              │
                └──────────────┘
```
其中：
- Operand A：操作数 A
- Operand B：操作数 B
- ALU Control：决定执行什么运算
- Result：运算结果

## ALU Control
---
ALU 控制信号 告诉 ALU 当前到底执行哪一种操作
至于 ALU Control 如何由 Instruction 解释而来 是由 **Control Logic** 来完成

## Adder 和 Subtractor
---
ALU中会包含 Adder 即加法器
Adder 可以通过 补码(Two's Complement)系统中 搭建 Subtractor 即减法器

由于 `A - B = A + B的补码 = A + ~B + 1`

于是让B 通过 XOR异或 进行控制
```
          Sub Control(接入Carry In)
              │
              ▼
B ───────→ XOR ──────→ Adder
A ───────────────────→ Adder
```

- Sub/Carry = 0 时 B xor 0 = B   于是 Adder 计算 `A + B`
- Sub/Carry = 1 时 B xor 1 = ~B 于是 Adder 计算`A + ~B + 1=A - B`

于是 加法和减法共享同一套加法器主体 是数字硬件设计中经典的资源复用

## Logic Unit
---
逻辑运算单元 主要执行逐位的逻辑运算 
例如
```
假设  A = 1010
	 B = 1100
	 
A AND B = 1000
A OR B = 1110
A XOR B = 0110
NOT A = 0101
```

## Comparator
---
比较器 用于判断
```
A == B
A != B
A < B
A > B
```
并不一定必须存在于 ALU 中 也可以存在独立比较逻辑

> 相等比较

即 `A==B`  通过 A XOR B 来实现
```
A XOR B
   │
   ▼
所有 bit 是否都是 0
   │
   ▼
Equal
```
当所有bit经过 异或后 都为0 那么Equal成立
即 `Equal = (A Xor B) == 0`

> 大小比较

首先 判断 A 和 B 是 unsigned 还是 signed

- Unsigned Compare 无符号比较
  直接按照二进制大小比较
- Signed Compare 有符号比较
  一般按补码解释 来比较

## SLT
---
常见的 RISC 风格的ISA中会出现
即 Set Less Than 小于则置位
概念上
```
if (A < B)
    Result = 1
else
    Result = 0
```
可能由 ALU 的比较逻辑完成
## Shift
---
即 移位
例如：
```
Logical Left Shift 逻辑左移
Logical Right Shift 逻辑右移
Arithmetic Right Shift 算术右移
```

移位功能的常见组织方式
- 直接放进ALU
- 做成独立的 Shifter 移位器 或者 Barrel Shifter 桶形移位器(作为 EU Execution Unit 执行部件的一个独立功能部件)

## Flag
---
有些ALU 除了产生 Result 还会产生一些 状态信息 即 Flag 标志

常见包括 :
- Zero Flag
- Carry Flag
- Overflow Flag
- Negative / Sign Flag

**不同 ISA 和 不同微架构 对 Flag 的使用方式 差异很大**

## Zero Flag
---
**零标志位** 表示 `Result == 0`
常用于 比较和条件判断

## Carry Flag
---
**进位标志** 主要与 Unsigned Arithmetic 无符号算术 有关
若 `A+B` 最高位产生额外进位 就会出现 Carry

## Overflow Flag
---
**溢出标志** 主要于 Signed Arithmetic 有符号算术 有关
主要用于判断 Signed Arithmetic 是否超出表示范围

例如 8 bit Two's Complement：
范围：`−128∼127`
计算：`127+1`
数学结果是：`128`
但 8 bit signed 无法表示。
结果 bit 会变成：
```
1000_0000
```
如果按 signed 解释就是：`-128`
于是：
```
Overflow = 1
```

## Negative/Sign Flag
---
**负数标志** 一般情况下 `Negative = Result[MSB]`

## 数据来源
---
ALU的输入可能来自：
- Register File
- Immediate 立即数
- PC
- Pipeline Register
- ==Forwarding Path==
- 某些内部临时值

数据来源 通常由外部 MUX 和 Control Logic 决定

## Execution Unit
---
执行单元 即CPU中真正完成某类运算的硬件单元
例如 : 
```
Integer ALU 整数算术逻辑单元
Shifter
Multiplier
Divider
Floating-Point Unit 浮点运算单元
Load/Store Unit
```

## Integer ALU
---
整数算术逻辑单元 现代CPU说 ALU时 默认指的就是整数ALU
它处理：
- Integer Add/Sub
- Bitwise Logic 按位逻辑
- Compare
- 某些简单 Shift
而浮点：
```
FP32
FP64
```
通常交给 FPU。
乘法和除法也可能有专门的：
```
Multiplier
Divider
```