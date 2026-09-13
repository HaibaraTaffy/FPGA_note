# 计算机组成原理学习目录

当前主线：

**体系结构基础 → RISC-V → AI 处理器**

## 00 计算机系统概览
保留原有计算机系统、软件层次和分类基础。

## 01 数据表示与可靠性
保留数值表示、字符编码、进制、BCD、校验等基础。

## 02 体系结构基础
这一部分只学习进入 RISC-V 前必须建立的处理器基础。

- `00 处理器整体结构`：后续放 CPU 整体结构、Processor State / Datapath / Control 的关系。
- `01 ISA 与 Microarchitecture`：ISA 与微架构。
- `02 Processor State`：Register、Register File、PC、IR。
- `03 Execution Hardware`：ALU、Shifter / Barrel Shifter；Comparator 后续并入这里，不再单开大章。
- `04 Instruction`：后续放通用 Instruction / Opcode / Operand / Immediate / 指令类型。
- `05 Datapath 与 Control`：后续放数据通路与控制通路。
- `06 CPU Execution Process`：保留旧教材式 CPU 工作过程，作为经典模型参考。
- `07 Pipeline`：后续放 Single-Cycle / Multi-Cycle / Pipeline 及基础 Hazard。
- `08 Memory Interface`：后续只放 CPU 访问存储器所需的接口概念。

## 03 存储系统与 I_O
从处理器基础中独立出来，避免把“存储器基础”和“CPU 内部结构”混在一起。

- `01 存储器基础`：现有《存储器》笔记。
- `02 Memory Hierarchy`：后续放 Cache、Locality、SRAM/DRAM、Latency/Bandwidth 等。
- `03 外设与 I_O`：现有外设笔记。

## 04 性能与评价
保留速度、时钟、性能指标等内容。

## 10 RISC-V
体系结构基础结束后进入。后续课程笔记放这里。

## 20 AI 处理器
RISC-V 与必要的微架构/存储基础完成后进入。后续长期主线放这里。

---

## 后续上课的固定规则

以后每节课开始前，先说明：

> **本节笔记放置位置：`具体文件夹 / 文件名.md`**

然后再开始授课。

如果本节属于已有笔记的补充，会直接说明应该追加到哪个现有文件，而不是随意新建笔记。
