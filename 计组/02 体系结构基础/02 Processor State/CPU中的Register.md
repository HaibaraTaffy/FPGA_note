寄存器  从RTL/微架构视角来看 它表示 **一组在时钟周期之间保留状态的位**

## CPU中的寄存器一览
---
```
CPU Register
│
├─ Architectural Register
│  ├─ General-Purpose Register
│  └─ Special-Purpose Register
│
└─ Microarchitectural Register
```

```
CPU Registers
│
├─ General-Purpose Registers
│
│  └─ 普通操作数、中间结果、地址
│
├─ PC
│  └─ 指令地址
│
├─ Status / Flag Register
│  └─ 条件、状态信息
│
├─ Control Register
│  └─ 处理器控制状态
│
├─ Stack Pointer
│  └─ 栈相关地址
│
├─ Link / Return Address Register
│  └─ 函数调用返回地址
│
└─ Microarchitectural Registers
   ├─ Pipeline Registers
   └─ 其他内部状态
```

## Architectural Register
---
体系结构寄存器 是 ISA明确定义 软件能够感知其存在和行为的寄存器
包括了 **General-Purpose Register** 通用寄存器 和 **Special-Purpose Register** 专用寄存器

例如 ISA 会规定：
- 有多少个通用寄存器；
- 每个寄存器多少 bit；
- 哪些指令可以访问它们；
- 哪些寄存器有特殊含义等
如果 ISA 规定有 32 个通用寄存器 那么程序中的 Instruction 就可以通过**寄存器编号**(R0 R1等)引用它们。

## GPR
---
即 **General-Purpose Register** 通用寄存器
用于保存程序运行过程中的普通数据 如一般操作数 中间结果 地址等程序数据
**但是Register本身并不区分是何种数据**

例如：
```
R1 = 5
R2 = 100
R3 = 某个地址
R4 = 某个中间结果
```

具体的ISA可能会对其中某些寄存器附加约定

## SPR
---
即 **Special-Purpose Register** 专用寄存器
这类寄存器主要承担某个特定体系结构功能

最典型的是 **PC** 即Program Counter 程序计数器 本质上还是Register
PC中保存的是 **当前 或 下一条待取Instruction的地址**

除了 PC，不同 ISA 还可能有其他特殊寄存器，例如：
- 状态寄存器；
- 控制寄存器；
- 异常相关寄存器；
- 栈指针；
- 链接寄存器。
## RF
---
即 Register File 寄存器堆
表示多个Register组织成一个统一的可寻址存储结构
但不是简单堆叠
它还必须有：
- Read Address；
- Read Data；
- Write Address；
- Write Data；
- Write Enable；
- 一个或多个读写端口。

## Microarchitectural Register
---
即微架构寄存器 为了实现某种具体微架构而存在 但不属于 ISA 对软件公开状态的寄存器

常见的是 **Pipeline Register** 流水寄存器
涉及 **微架构状态** 而不是 **体系结构状态**

## 体系结构状态
---
Architectural State 即**从ISA角度看 程序执行到某个时刻时 软件可观察到的处理器状态**

常见的 :
```
PC
Architectural Registers
部分状态/控制寄存器
Memory State
```