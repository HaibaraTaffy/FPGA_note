# FPGA Note

## 建议使用 Obsidian 打开文件，搭配 Minimal 主题以获得最佳观看体验

## 建议阅读顺序

这份仓库目前有两条可以互相交叉的学习主线：

- **FPGA 工程主线**：FPGA 资源与基础 → 协议与数据传输 → 外设与系统设计
- **处理器主线**：数据表示 → 体系结构基础 → RISC-V → RV32I Mini Core → AI 处理器

如果你是刚接触 FPGA，建议先完成一个最小工程，再从第一部分开始阅读。计算机组成、RISC-V 与 AI 处理器部分仍在持续整理中。

### 1. FPGA 资源与基础

> 我个人强烈建议先上手写过一个工程，无论是点亮 LED 还是更复杂的工程。只有真正跑到板子上、熟悉基本流程之后，才能更好地吸收后面的内容。
>
> 在具备基础数电知识后，从最底层的资源开始，理解每一行 HDL 最终会映射成什么硬件、约束为什么这样写、资源占用率意味着什么，以及组合逻辑为什么由 LUT 实现。先建立整体认识，再进入复杂工程，就不容易只知其然而不知其所以然。

1. `Logic Cell Slice CLB与FPGA Fabric`：先认识 FPGA 的整体结构。
2. `LUT` → `FF` → `MUX与Routing` → `Carry Chain`：理解基本逻辑、寄存器、布线与算术进位资源。
3. `DIstributed RAM 与 SRL` → `BRAM` → `DDR` → `FIFO`：学习片上、片外存储与数据缓冲。
4. `DSP`：了解乘加等专用算术资源。
5. `IO与BANK` → `Clock Network` → `PLL与MMCM`：理解外部接口、时钟网络、复位与基础时序分析。
6. `DDRIO 和 SERDES` → `High-Speed Transceiver 和 Hard IP`：进入高速接口与硬核资源。
7. `Timing Analysis`：建立时序约束与时序收敛意识。

### 2. 数据表示与可靠性

进入处理器前，建议先补齐数据在硬件中的表示方式：

`进位计数法、进制转换与 BCD 码` → `数值` → `字符与字符串` → `校验原理、奇偶校验码、海明码与 CRC`

### 3. 体系结构基础

这一部分按照“先建立全局图景，再拆解处理器模块”的顺序阅读：

1. `计算机分类与发展方向` → `计算机系统的组成` → `计算机系统的层次结构` → `软件系统`
2. `CPU整体结构` → `ISA与微架构`
3. `CPU中的Register` → `PC` → `IR` → `RF Register File`
4. `ALU` → `Shifter / Barrel Shifter`
5. `Instruction 基础` → `Instruction Class` → `Instruction Format`
6. `Datapath` → `Control` → `CPU及工作过程` → `Single-Cycle CPU`

完整目录和各部分定位可参考 [`计组/README.md`](计组/README.md)。

### 4. RISC-V 与 RV32I Mini Core

完成体系结构基础后，可以沿着仓库中新整理的 Mini Core 笔记，把概念落实到一颗真正的单周期 CPU：

1. [`总体设计`](计组/10%20RISC-V/RV32I%20Mini%20Core/总体设计.md)：了解当前 CPU 的目标、模块划分和存储器模型。
2. `PC Register` → `Instruction Memory` → `Register File`：先搭建处理器状态与取指基础。
3. `ImmGen` → `ALU`：建立立即数生成和执行单元。
4. `Main Decoder` → `ALU Decoder`：完成控制信号译码。
5. `Branch 与 Next-PC Logic`：处理顺序执行与分支跳转。
6. `Data Memory` → `Top-Level Datapath`：接入访存并连接完整数据通路。

当前第一阶段实现 `add`、`sub`、`and`、`or`、`xor`、`addi`、`lw`、`sw`、`beq` 九条 RV32I 指令。具体 RTL 代码见 [`RV32I_Mini_Core_v1`](https://github.com/HaibaraTaffy/RV32I_Mini_Core_v1)。

### 5. 存储系统、I/O 与性能

完成单周期 CPU 基础后，再阅读：

`存储器` → `Memory Hierarchy` → `外设与 I/O` → `速度`

这部分用于继续理解存储层次、处理器与外设的边界，以及延迟、带宽和性能评价。

### 6. 协议与数据传输

先阅读 `通信概念`，再按需学习 `I2C`、`USB协议`、`以太网与 UDP 通信基础`；之后阅读 `分辨率`、`内存读写方式` 和 `跨时钟域CDC`。

### 7. 外设与系统设计

建议按照 `OV5640` → `时钟复位和跨时钟` → `视频流、DDR缓存与切换` 的顺序，将基础知识带入完整数据通路。

### 8. AI 处理器

这一部分是后续长期主线。建议先完成 RISC-V、基础微架构和存储系统，再进入 AI 处理器架构学习。本人还未进入学习 敬请期待

### 9. 工具与项目记录

`其他/` 中的 Vivado、PDS、Primitive、综合、布局布线及项目记录相对独立，可以在实际开发遇到对应问题时查阅。










