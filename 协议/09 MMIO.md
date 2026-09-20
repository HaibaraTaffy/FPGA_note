即 **Memory-Mapped I/O** 内存映射输入输出
是一种**地址映射方式/访问机制**
并非一个具体模块

## 基本思想
---
**将外设寄存器 分配到 CPU的地址空间中**
于是 CPU 可以像访问内存一样 用普通的Load/Store 指令访问外设
回答**外设怎么出现在CPU地址空间里**

## CPU内的地址空间
---
可以先学习 [mynote/计组/03 存储系统与 I_O/01 存储器基础/00 存储器]
先建立一个很重要的认识 **CPU的地址本身只是编号 这个编号对应的是 内存 还是某个外设 由系统的地址映射决定**

例如 系统可以规定:
```
0x0000_0000 ~ 0x1FFF_FFFF
        ↓
       RAM
0x4000_0000 ~ 0x4000_FFFF
        ↓
      UART
0x5000_0000 ~ 0x5000_FFFF
        ↓
      GPIO
0xA000_0000 ~ 0xA000_FFFF
        ↓
    PCIe FPGA BAR
```
在CPU眼中 并不区分这些都是什么 只知道是地址

## MMIO的功能
---
直接上例子

假设 UART 有一个数据寄存器
系统规定：
```0x4000_0000 =UART_DATA```
那么 CPU 执行：
```Load 0x4000_0000```
CPU看起来是在 “读一个内存地址”
但实际上**硬件**会发现：
```这个地址属于 UART```
于是请求不会去 RAM
而是去：
```
CPU
 ↓
Bus / Interconnect 总线互联
 ↓
Address Decoder 地址解码器
 ↓
UART 串口
 ↓
UART_DATA 串口数据
```
同样CPU执行：
```Store 0x4000_0000```
就是在写 UART_DATA
这就是大名鼎鼎的 **Memory-Mapped I/O**

## Address Decoder
---
系统中会有地址译码逻辑
可以粗略理解成：
```
                 CPU
                  │
                  │ Address
                  ▼
          ┌───────────────┐
          │ Address Decode│
          └───────┬───────┘
                  │
       ┌──────────┼──────────┐
       │          │          │
      RAM        UART       GPIO
```

这是实现 **MMIO** 的重要硬件基础
在现代 SoC 中 这个功能通常由
```
AXI Interconnect
NoC
Bus Fabric
Root Complex
```

## Register Map
---
即 寄存器映射表
由于外设一般需要很多控制信息 数据量小 没有必要做复杂的数据流接口
**一个地址对应一个寄存器** 非常好

再加上我们的 `Base Address+ Offset` 格式 可以很方便的读写寄存器
例如：
```Base Address = 0x8000_0000```
那么：
```
Base + 0x00
→ CTRL
Base + 0x04
→ STATUS
Base + 0x08
→ MODE
Base + 0x0C
→ LENGTH
```
这种表 就是 **Register Map**

## MMIO与RAM的重要区别
---
**MMIO 地址虽然像内存 但背后的硬件行为 不一定像普通RAM** 
MMIO地址 映射的是外设寄存器 **有被CPU外的硬件修改的风险**
对应上我之前单片机开发的 `volatile`
这里的 `volatile` 在提醒编译器 : "这个地址背后的值可能随时被硬件改变 不要随便优化掉访问"