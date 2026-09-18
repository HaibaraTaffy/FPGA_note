时钟网络 本身也是FPGA的专用物理资源 并不是抽象的一条道路

## Clock Fanout
---
时钟信号是一个典型的高扇出信号(High-Fanout Signal)
控制着大量时序单元在什么时候采样数据
所以我们一般不会用普通逻辑去生成Clock 无法满足高扇出 低偏差的要求

## Clock Skew
---
如果走普通可编程Routing 会因为路径长度不同的原因 导致时钟沿到达时间不同 最终产生 **Clock Skew(时钟偏差)**

即**同一个时钟沿到达不同寄存器的时间差**

理想情况 是让Skew -> 0
但是做不到 所以只能尽量控制
由于存在到达时间差 影响采样时机
所以 **Clock 到达时间差 会改变数据路径的时序裕量**

## Dedicated Clock NetWork
---
专用时钟网络 专门为
- 高 Fanout
- 低 Skew
- 大范围分发
设计的

                   Clock Source
                        │
                        ▼
                 ┌────────────┐
                 │Clock Buffer│
                 └──────┬─────┘
                        │
               Dedicated Clock
                    Network
                        │
          ┌─────────────┼─────────────┐
          ▼             ▼             ▼
        Region        Region        Region
          │             │             │
       FF/DSP         FF/BRAM        FF...

## Clock Buffer
---
时钟在进入专用Clock Network之前 先要经过专门的`Clock Buffer`
在Xilinx中 即BUFG(Buffer global) 可以理解为 `Global Clock Buffer(全局时钟缓冲)`

## Global Clock
---
全局时钟网络 用于把时钟分布到FPGA的大范围区域
目标是 **尽可能让整个器件中的大量时序资源获得质量良好的时钟**

除了全局 Clock Network 还可能存在：
- Regional Clock
- Local Clock
- I/O Clock
等不同层级

## Clock来源
---
- 板上晶振
- 外部
- PLL/MMCM

比较典型的一个架构是
```
External 50MHz
      │
      ▼
   PLL/MMCM
      │
      ├──► 100MHz
      ├──► 200MHz
      └──► 25MHz
```

## 专用IO
---
**FPGA的外部引脚并不完全一样** 
一些引脚具有 `Dedicated/Clock-Capable Input`
能够更直接地进入专用的Clock Network
于是 在板级设计中 外部晶振就会连接到`Clock-Capable Pin`


## Clock Gating
---
时钟门控 即**不需要工作时 关闭某部分时钟 以降低动态功耗**
可能会以为
```Verilog
assign gated_clk = clk & enable;
```
但FPGA中不能这么干  会产生`Glitch(故障)` 
如果需要用Clock Gating 则需要使用FPGA提供的专用 Clock Enable/Clock Buffer 资源

## Clock Domain
---
由不同时钟驱动的逻辑 属于不同的**时钟域(Clock Domain)**
在不同时钟域之间传递数据 称为 **CDC(Clock Domain Crossing 跨时钟域)**
在CDC中 由于两个时钟的边沿可能无固定关系 所以会涉及到 **Setup/Hold Violation(建立/保持 违例)** 进而可能导致 **Metastability(亚稳态)**
Async FIFO是解决CDC的一个好方法

## Clock Enable
---
不能随意将普通逻辑输出 当做新的全局输出 可以利用
```
PLL/MMCM
Dedicated Clocking Resource
Clock Buffer / Divider
```
若只是希望**某部分逻辑** 每两个周期工作一次 可以使用 **Clock Enable**
例如
```
always @(posedge clk) begin
    if (ce)
        q <= d;
end
```
该逻辑块时钟由原时钟驱动 **只是在时钟有效边沿到来时 只有 CE 有效才允许寄存器更新状态**
当目标只是让某段时序逻辑“降低更新频率”时 使用 Clock Enable 这个方法 通常比用普通逻辑生成一个低速 Clock 更好
Clock Enable没有制造出新的Clock Domain

## Clock Frequency
---
时钟频率
我们可以要求一个目标时钟频率(外部接口需求或者内部设计) 但是在这个时钟频率下的逻辑 都需要在物理上满足一定的时序约束 例如建立时间和保持时间 所以这是一个寻找平衡点的过程 性能与时序裕量之间的权衡
- 更高的时钟频率 意味着更加紧张的时序裕量 
- 而更低的时钟频率 则可能降低系统的吞吐能力
因此 目标的Clock Frequency 不能脱离实际的数据路径延迟和器件性能来确定

Clock Network无法提高Clock Frequency 只能说能有助于稳定Clock 将Clock以**低 Skew 高 Fanout 可控延迟**的方式分发出去 至于Clock的频率上限 还需要看其他时序的约束是否可以满足
另外 提高频率不一定提高最终性能 吞吐量 位宽 利用率等都会影响最终的性能

