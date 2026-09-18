部分FPGA架构允许LUT不仅作为组合逻辑使用 还可以配置为:
1. DIstributed RAM (分布式RAM)
2. SRL (Shift Register LUT) 移位寄存器

对LUT更深的理解 : **LUT 是一种可配置逻辑资源 其中部分 FPGA 架构允许 LUT 内部资源工作在 Logic Distributed RAM SRL 等不同模式**

## DIstributed RAM
---
某些LUT内部的存储结构允许在运行过程中写入

于是:
![[FPGA_Distributed_RAM_SRL.svg]]

这个时候LUT是作为 **小容量RAM** 
这种LUT 统称DIstributed RAM(分布式RAM)

这种RAM 适合用于容量小(几十bit到几kbit) 靠近逻辑的数据存储
典型用途:
- 小型查找表
- 小缓存
- 小型 RAM
- 寄存器文件
- 小型 FIFO

对于小型的存储器 综合器可能会推断出DIstributed RAM

## SRL
---
Shift Register LUT 移位寄存器
在打拍的时候用的多 即Delay Line
SRL擅长规则 连续的移位延迟