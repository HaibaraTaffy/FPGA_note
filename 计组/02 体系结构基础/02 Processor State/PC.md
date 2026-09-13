**CPU内部 专门保存 "下一步从哪里取指令" 的地址寄存器
Program Counter** 程序计数器

## 本质
---
本质上就是一个寄存器
提供取指地址

存放 **Instruction Address 即指令地址**
例 `PC = 0x0000_1000` 意为 `CPU要去地址 0x1000 的位置取Instruction`
然后 **存储系统**根据这个地址 返回 `Instruction Data`

PC内存放的地址 是不断变化的
一般来说 `PC_next = PC + 当前 Instruction 的长度`
我们这里以 Instruction 长度为 4 来说明更加具体

## 更新
---
PC 是 Register，所以它不是组合逻辑输出一变化就立刻改值。
假设：
```
Current PC = 0x1000
```
组合逻辑先计算：
```
Next PC = 0x1004
```
在这个阶段：
```
PC Q = 0x1000
PC D = 0x1004
```
当前 PC 仍然是：
```
0x1000
```
到了有效 Clock Edge：
```
PC <= Next PC
```
之后才变成：
```
PC = 0x1004
```
这和普通同步 Register 完全一样。

## Current PC 和 Next PC
---
- Current PC 即当前PC 指的是 PC Register 当前Q端保存的值
- Next PC 即下一PC 指的是 准备在下一次PC更新时 写入PC的值

>注意 PC中的地址 不一定是递增的

由于
- Branch — 分支
- Jump — 跳转
- Function Call — 函数调用
- Return — 返回
- Exception — 异常
- Interrupt — 中断
的存在 会改变 `Instruction Flow`

## Next_PC Logic
---
下一PC生成逻辑 负责计算或选择下一个PC
概念图
```
                 PC + Instruction Length
                          │
                          │
Branch Target ────────────┤
                          │
Jump Target ──────────────┤
                          ▼
                 ┌────────────────┐
                 │ Next-PC Logic  │
                 └───────┬────────┘
                         │
                         ▼
                      Next PC
                         │
                         ▼
                         PC
```

>顺序执行时

使用加法器 进行地址更新
```
PC.Q = 0x1000
      │
      ▼
 Adder 加法器
      ▲
      │
      4
      │
      ▼
0x1004
      │
      ▼
Next PC
      │
Clock Edge
      ▼
PC.Q = 0x1004
```

PC + 4不一定非要占用主ALU
很多微架构会单独提供用于PC更新或地址计算的加法逻辑 Adder

>遇到 Branch 时

当前 Instruction 是条件分支 且条件成立
那么会进行跳转 跳转到 Branch Target 即分支目标地址

所以此时有两个候选：
```
Sequential Address = 0x1004
Branch Target      = 0x2000
```
必须二选一 需要MUX
MUX的选择信号 被称为**Branch Taken 分支成立/分支被采用**
表示当前条件分支决定改变正常顺序路径

若 Branch Taken = 0 则选择PC + 4
若 Branch Taken = 1 则选择 Branch Target 

## Jump 和 Branch
---
- Branch 分支 通常指的是 根据某个条件决定是否改变控制流(if else)
- Jump 跳转 通常指的是 无条件改变控制流

所以PC的候选来源 :
```
PC + 4
Branch Target
Jump Target
```

## Control Flow
---
控制流 指 程序中的Instruction 按什么顺序被执行
**PC的变化 直接决定 CPU下一步执行程序的哪个位置**
PC是整个处理器控制流的核心状态之一

## PC和MAR
---
- MAR : Memory Address Register 存储器地址寄存器

传统的教学模型 中 PC提供一个Memory Address 再由MAR保存地址
但是现代CPU 实际微架构并不一定存在一个 独立 统一的MAR的硬件寄存器
现代处理器通常会有很多：
- 地址寄存器；
- 流水寄存器；
- 地址生成逻辑；
- Cache 接口寄存器。

## 复位
---
CPU复位之后 不能让PC是随机值 否则CPU根本不知道从哪里开始执行程序 一般系统会规定一个 `Reset Vector`
- Reset Vector : 复位向量 表示 CPU Reset结束后 开始取指的规定地址 

 然后CPU从这个位置开始执行启动代码 具体的Reset Vector地址取决于处理器和系统设计

## Branch Prediction
---
即 分支预测 
在程序中 遇到Branch 而CPU暂时还不知道Branch是否成立 就会出现 **下一条到底取哪条Instruction?** 的问题
于是就引出了 分支预测

## RTL实现
---
```Verilog
always @(posedge clk) begin
    if (reset)
        pc <= RESET_VECTOR;
    else
        pc <= next_pc;
end
```
