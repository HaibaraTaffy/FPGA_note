## Logic Cell
---
**逻辑单元** 可以理解为FPGA中较小颗粒度的逻辑资源组合
核心组成通常包括:
- LUT
- FF
- MUX
- Carry 相关逻辑
- 一些局部连接资源

>注意 Logic Cell 并不是所有 FPGA 厂商统一定义的物理结构名称
>不同厂商、不同架构的术语和具体组成不同

主要是LUT和FF放在一起 可以通过内部的局部连接快速传输

## Slice
---
多个LUT FF以及相关资源 可以进一步组成一个更大的逻辑块 即Slice
在Slice中 还有专门的局部连接 于是更加高效

## CLB
---
**可配置逻辑块**(Configurable Logic Block)
在 Xilinx 架构中可粗略建立 CLB、Slice、LUT/FF/Carry/MUX 的层次关系；具体组成因器件系列而不同。

![[FPGA_Fabric_层次.svg]]
## FPGA Fabric
---
可以理解为 FPGA 中由 `可编程逻辑资源 + 可编程互连资源`形成的主体逻辑区域。

> 同一张图同时展示了 CLB/Slice 层次与 Fabric 中的逻辑、BRAM、DSP、可编程互连关系。
