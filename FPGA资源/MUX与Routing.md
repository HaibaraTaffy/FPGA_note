## MUX
---
Multipler 多路选择器
FPGA内部大量资源本身就是通过MUX实现"可配置"的
通过配置MUX的选择状态 可以决定资源之间的连线

## Routing
---
即布线/互连
负责连接FPGA内部的不同资源 例如LUT -> FF

可以粗略分为三类
- Wire : 实际的金属连线
- Switch/Switch Box : 控制某两段连线是否连接
- MUX 从多个可能来源中选择一个

在FPGA配置时,不仅LUT的真值表被确定了 Routing的连接关系也会被确定
这些信息都是包含在bitstream中的

Routing还可以分为
- Local Routing : 延迟小 资源开销低 常在Slice中
- General/Global Routing : 跨较远逻辑区域的可编程布线 常在Slice 间

Routing也会有传播延迟 且会占很大比例 
Placement 会直接影响 Routing Delay 于是RTL完全相同的情况下 不同的布局布线结果 最高频率也可能不同

## Fanout
---
扇出 表示一个信号驱动多少个负载
高扇出会带来
- 更大的负载
- 更复杂的Routing
- 更大的延迟

像 `Clock Reset Enable` 这些高扇出信号通常是需要特别处理的

Clock会使用专门的Clock Network

## Implementation
---
实现 就是 Place&Route的总称
