## 基本定义
---
Programmed I/O 程序控制输入输出
**首先是一种 数据传输方式 控制思想 并非一个固定的硬件模块**

**PIO = CPU 亲自通过指令读写外设寄存器来完成数据传输**
**本质上 CPU利用普通 Load/Store 指令直接访问外设寄存器**
 
常见两种工作方式 : `Polling` `interrupt`
## Polling
---
即 轮询 CPU不断询问 

## Interrupt
---
即 中断 CPU不需要一直询问 而是通过 Interrupt 进入 `ISR(Interrupt service routine 中断服务程序)` 再进行处理

```
CPU正常干活
↓
UART收到数据
↓
UART发Interrupt
↓
CPU进入ISR
↓
CPU读取UART_DATA
```

## 与 DMA 比较
---

| 概念           | PIO            | DMA                  |
| ------------ | -------------- | -------------------- |
| 全称           | Programmed I/O | Direct Memory Access |
| 本质           | 数据传输方式         | 数据传输机制               |
| 谁搬数据         | CPU            | DMA硬件                |
| CPU是否逐数据参与   | 是              | 否                    |
| 是否一定需要专用硬件模块 | 否              | 通常需要                 |
| 大数据效率        | 较低             | 高                    |
| 实现简单程度       | 很简单            | 更复杂                  |
| 常用于          | 少量控制数据         | 大批量数据                |