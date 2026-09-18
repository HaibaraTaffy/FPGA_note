**FIFO其实是一个逻辑结构 而不是真实存在的物理资源**

## 概念
---
FIFO : 即First In First Out **先进先出**
但实际上FIFO并不是让所有数据真的一级一级移动
而是通过维护两个指针 Read Pointer 读指针 和 Write Pointer 写指针 来实现先进先出

> FIFO=Memory+Write Pointer+Read Pointer+Control Logic​

## 写指针 读指针
---
写指针 : `wr_ptr` 表示下一次数据应该写到哪
读指针 : `rd_ptr` 表示下一次该从哪里读取数据

## 本质
---
fifo本质上是一个 **Circular Buffer** 即环形缓冲区
FIFO还是一个 **受规则约束的 顺序数据流接口**
- 与RAM区别 
  RAM可以通过用户提供地址 随机访问任意地址的数据
  而FIFO只有数据接口和使能 有些有标志满和标志空 内部由指针自动管理地址

## Full Empty Almost Full Almost Empty
---
标志满 : 避免多写覆盖原先数据
标志空 : 避免快读读出无效数据
快满标志 : 提前告诉上游 快满了 提前留余量 起到流控作用
快空标志 : 提前告诉下游 快空了 流控

## 典型接口
---
![[FPGA_FIFO_结构.svg]]

## 解耦
---
可以将生产者与消费者解耦 做到一个较为宽松的缓存作用
FIFO可以吸收生产者与消费者之间的短期速率波动
**但不能解决长期带宽不足的问题**

## 实现
---
多考虑使用 Simple Dual-Port RAM 来实现
FIFO需要读写两个端口

## 同步 异步FIFO
---
- 同步FIFO(Synchronous FIFO) : `wr_clk = rd_clk` 读写共用一个时钟
- 异步FIFO(Asynchronous FIFO/Dual-Clock FIFO) : 读写不共用一个时钟 可以用来做CDC
用异步FIFO来做CDC是非常常见的方法

## FIFO的延迟
---
FIFO可能由BRAM实现 若BRAM存在同步读延迟 则FIFO的输出自然也会受到影响

## FIFO模式
---
平时使用FIFO的情况 大部分是需要配置IP核 一些IP核会提供
```
Standard FIFO
FWFT FIFO
```
Standard FIFO : 具有读延迟 当有效信号到来时 等待Latency 输出数据
FWFT FIFO : 具有预取机制 FIFO非空时 第一个可读数据已经出现在dout上


