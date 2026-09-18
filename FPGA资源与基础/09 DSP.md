**专门用于高速数值运算的硬件计算单元** (取名来源 Digital Signal Processing)
是FPGA内部一种**专用算术硬件块(Block)** 在芯片设计时已经制造好的专用硬件资源

## MAC
---
将DSP Block抽象
 A ─────┐
        │
        ▼
    ┌────────┐
 B ─► MULT   │
    └───┬────┘
        │
        ▼
    ┌────────┐
 C ─► ADD    │
    └───┬────┘
        │
        ▼
        P

`P = A X B + C`
即MAC Multiply-Accumulate 乘加运算

实际上还可能包含
```
Pre-Adder
Multiplier
Adder / Accumulator加法器/累加器
Registers
Cascade Logic级联逻辑
MUX
```


虽然用LUT也可以实现乘法 但是需要消耗的资源很多 且还会带来时序压力和功耗的增加

## MAC的重要性
---
大量的数字计算 都可以拆成 `A X B + C`的形式
例如FIR y = ∑x_i X h_i
矩阵乘法 C_ij = ∑A_ik X B_kj
神经网络 y = ∑w_i X x_i + b

因此 DSP Block 天生适合：
- 数字信号处理
- 图像处理
- 矩阵计算
- 神经网络
- AI 加速

## DSP内的Pipeline
---
由于为了运行在较高频率 在DSP内部也会提供Pipeline Register
```
A/B
 │
 ▼
Register
 │
 ▼
Multiplier
 │
 ▼
Register
 │
 ▼
Adder
 │
 ▼
Register
 │
 ▼
P
```
一个复杂的乘加操作就会被切成多个Pipeline Stage

虽然会引入更多的Latency 但是流水线填满后 可以做到 每个Clock 接收一组新输入 于是
`Latency ≠ Throughout(吞吐量)`

## Latency和Throughout
---
假设 DSP：
```
Latency = 3 cycles
```
输入：
```
Cycle 0 → Data A
Cycle 1 → Data B
Cycle 2 → Data C
Cycle 3 → Data D
```
输出可能：
```
Cycle 3 → Result A
Cycle 4 → Result B
Cycle 5 → Result C
Cycle 6 → Result D
```
所以单个数据：Latency = 3 cycles
但是当流水线填满后 Throughout = 1 result/cycle

## DSP Cascade
---
很多FPGA的DSP Block之间还有专用的 **级联路径**
可以降低时序压力 适合∑A_i X B_i 这种连续MAC结构

## DSP Inference
---
乘法可能识别成DSP 但是也不一定
有时非常小的乘法 用LUT反而更合理
也可以通过IP进行配置
当然 原语也是可以的

## DSP的位宽
---
DSP的位宽是有限的 若是超过了 可能需要进行**多个DSP拼接**
于是 数据位宽会直接影响DSP资源的消耗

## PE
---
Processing Element 即处理单元
一个PE可以抽象成
          PE

Input A ─┐
         ▼
        MAC
         │
Input B ─┘
         │
         ▼
    Accumulator
         │
         ▼
       Output

然后可以组成阵列 
```
PE PE PE PE
PE PE PE PE
PE PE PE PE
PE PE PE PE
```
形成 PE Array
