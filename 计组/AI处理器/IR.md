即 Instruction Register 指令寄存器
**是经典计组模型里一个非常重要的概念 但真实现代CPU不一定有一个唯一 独立的 称为IR的物理寄存器**

## 定义
---
IR 保存当前正在被解释或执行的 **Instruction**
比如 Memory 返回一条 Instruction：
```
ADD R3, R1, R2
```
从逻辑上，可以理解为先进入：
```
      Instruction Data
            │
            ▼
       ┌──────────┐
       │    IR    │
       └──────────┘
            │
            ▼
        Decode Logic
```
也就是：
**Memory 取回指令 -> IR保存 -> Decode使用**

## 本质
---
概念上：
```
┌────────┬────────┬────────┬────────┐
│ Opcode │  rs1   │  rs2   │   rd   │
└────────┴────────┴────────┴────────┘
```
这整个 bit pattern 就是一条 Instruction Encoding。
IR 保存的就是： 这一整条 **Instruction Encoding**

## Opcode
---
即 Operation Code 操作码
表示当前 Instruction 要执行哪种操作

实际上 `IR[某些位]` 就可能对应Opcode 也就是一串二进制码
然后 Decode Logic 就可以根据这些 bit 
把一条Instruction 拆成不同的信息 例如
```
IR
 │
 ├── Opcode
 ├── Register Fields
 ├── Immediate Fields
 └── Other Control Fields
```

## Instruction Format
---
指令格式 即一条 Instruction 的不同 bit分别表示什么
一般 Instruction Format 也会存在IR中
例如一个纯概念格式：
```
31          24 23       16 15        8 7         0
┌────────────┬────────────┬────────────┬────────────┐
│   Opcode   │    rs1     │    rs2     │     rd     │
└────────────┴────────────┴────────────┴────────────┘
```
那 IR 里就同时保存：
- Opcode 操作码
- Source Register Index 源寄存器索引
- Destination Register Index 目标寄存器索引
- Immediate 立即数
- Function Bits 功能位
- 其他 ISA 定义字段
具体有哪些字段 要看 ISA

## Instruction Decode Logic
---
指令译码逻辑 负责解释 Instruction
关系可以写成：
```
             IR
              │
      ┌───────┼────────┐
      ▼       ▼        ▼
   Opcode   rs/rd   Immediate
      │       │        │
      └───────┼────────┘
              ▼
        Decode Logic
```
IR只负责保存 不负责解释 
**IR是取指 和 译码之间的桥梁**

## 为什么现代CPU不一定有唯一一个IR
---
在现代流水线 CPU 中，一条 Instruction 往往会经过多个 Stage。
比如以后会学到：
```
Fetch
↓
Decode
↓
Execute
↓
Memory
↓
Write Back
```
这些阶段之间通常有：
**Pipeline Register — 流水寄存器**
例如：
```
Fetch
  │
  ▼
[Pipeline Register]
  │
  ▼
Decode
```
那么从 Fetch 得到的 Instruction Encoding，可能直接保存在这个流水寄存器里。
这时它在功能上已经承担了：
**保存待译码 Instruction**
这个 IR 的功能。
但这个寄存器物理上可能叫：
```
IF/ID Register
Fetch Queue Entry
Instruction Buffer
```
等等。
所以不一定有：
```
IR
```
这个唯一命名的寄存器。

## IR 属于 Microarchitecture State
---
服务于处理器内部实现 不暴露给ISA

## 位宽
---
IR的位宽不一定等于CPU位宽
要区分 **Data Width 和 Instruction Width**

## Instruction Width
---
指令宽度 指一条Instruction Encoding 占多少bit
例如`32 bit Instruction` 表示一条编码 4个字节
但不是所有的ISA 都固定长度
有些 ISA：
- 固定长度
- 有多种长度
- 可变长度(更加复杂 涉及指令完整性判断)
因此 IR 或相关 Instruction Buffer 的结构也会不同

## Instruction Buffer
---
指令缓冲区 不是只保存一条 Instruction 而可能保存多条 已经取回的指令或原始指令字节
例如：
```
Instruction Buffer
┌─────────────────────┐
│ Instruction A       │
│ Instruction B       │
│ Instruction C       │
└─────────────────────┘
```
区别 IR : IR更像单条当前指令状态 而 Instruction Buffer 是更一般的缓冲区

## 预取指
---
存在意义 : 为了让CPU硬件利用率提升 现代处理器 通常会让 Instruction Fetch 提前取得更多 Instruction
于是需要
```
Buffer
Queue
Pipeline Register
```
来保存它们

## RTL 实现
---
```Verilog
always @(posedge clk) begin
    if (ir_we)
        ir <= instruction_data;
end
```
需要WE
因为不是每个Clock edge都一定有一条新的有效 Instruction 可以写入
例如某些简单多周期处理器里：
- 只有 Fetch Cycle 才更新 IR
- 其他周期保持当前 Instruction

尤其常见于 多周期 CPU中

## Pipeline Register
---
流水线寄存器 保存两个Pipeline Stage之间需要传递的一整组状态 可能包含
```
Instruction
PC
Operand
Immediate
Control Information
```

现代微架构更喜欢 "成组保存阶段状态 而不是只保存 Instruction"

## Valid Bit 有效位
---
表示 当前这个寄存器/缓冲顶里的数据 是不是一条有效信息
- Valid = 1 表示有有效Instruction
- Valid = 0 表示无有效Instruction

## 与ISA的关系
---
ISA规定了 Instruction 的格式
IR保存 Instruction
Decode Logic 按照 ISA 去解释 Instruction
```
ISA
 ↓ 规定格式
Instruction Encoding
 ↓ 被保存
IR / Instruction Buffer
 ↓
Decode
```