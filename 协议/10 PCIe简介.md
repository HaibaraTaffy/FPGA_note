即 Peripheral Component Interconnect Express 
外围组件互连快件
## 基本概念
---
- Gen4/3/2/1 : 指的是 PCIe 的版本或代数
  - Gen1 : 最初 每通道支持`2.5GT/s` 的数据传输速率 有效数据传输率约为 `250 MB/s`
  - Gen2 : 翻倍 `5GT/s` 有效数据传输率约为 `500 MB/s`
  - Gen3 : 进一步提升到 `8GT/s` 有效数据传输率约为 `985 MB/s(使用 128b/130b 编码)`
  - Gen4 : `16GT/s` 有效数据传输率约为 `1969 MB/s`
  - Gen5 : `32GT/s` 有效数据传输率约为 `3938 MB/s`
    GT : Giga-transfers per second 千兆转移 以上均是每通道的传输速率

- x1/x2/x4/x8/x16 : PCIe 设备使用的物理通道数量(Lanes) 通道越多 理论上可以达到的总带宽就越高
  - x1 : 单通道 适合声卡 网卡 低带宽需求
  - x2 : 双通道 适合稍高带宽应用
  - x4 : 四通道 适合存储控制器或其他中等带宽需求
  - x8 : 八个通道 常见高端显卡
  - x16 : 最吊的 特别用于高性能显卡 能够提供最大带宽

**Gen 决定了每个通道的传输数据速率 xN决定了有多少个这样的通道并行工作 二者共同决定了最终的PCIe设备性能**

## 带宽计算
---
一般 带宽与 通道数量以及每通道数据传输速率 有关 可以计算

`总带宽 = 每通道的有效数据传输率 * 通道数量`

例如 我在集创赛中使用的 PCIE Gen2 x2 带入公式计算
`total_Bandwidth = 500 * 2 = 1GB/s`

## 总线架构
---
与 以太网的OSI模型类似 是一种**分层协议架构**
分为 :
- 事务层(Transaction Layer)
- 数据链路层(Data Link Layer)
- 物理层(Physical Layer)

这些层 都有 `rx`和`tx` 功能
![[Pasted image 20260920172855.png]]

**PCIe 体系结构中 使用数据包在设备之间传递信息 数据包在事务层和数据链路层中形成 以将信息从发送设备传到接收设备**

总体框架
```
上层逻辑 / DMA / BAR / 寄存器
            │
            ▼
┌─────────────────────────┐
│ Transaction Layer       │
│ 事务层                  │
│                        │
│ 我要读还是写            │
│ 读哪里                  │
│ 写哪里                  │
│ 数据是什么              │
└────────────┬────────────┘
             │ TLP
             ▼
┌─────────────────────────┐
│ Data Link Layer         │
│ 数据链路层              │
│                        │
│ 加序号                  │
│ 加校验                  │
│ ACK / NAK               │
│ 出错重传                │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│ Physical Layer          │
│ 物理层                  │
│                        │
│ 串并转换                │
│ Lane                    │
│ 差分信号                │
│ 链路训练                │
└────────────┬────────────┘
             │
       PCIe TX / RX
             │
             ▼
        对端 PCIe 设备
```

## Transaction Layer
---
负责描述一次 PCIe 访问到底想干什么
比如
```我要向地址 0x1000 写 32 Byte```
事务层会将这种请求封装成 `TLP`
**TLP Transaction Layer Packet 事务层数据包**
是PCIe中真正承载读写请求和数据的主要数据包

## TLP
---
大致理解一个TLP包
```
┌───────────────────────┐
│ Header                │
│                       │
│ 我要干什么             │
│ 地址是多少             │
│ 长度是多少             │
│ 谁发的                │
│ Tag是多少             │
├───────────────────────┤
│ Data                  │
│                       │
│ 实际数据               │
└───────────────────────┘
```
不是每个TLP都有Data

## Transaction
---
事务 一次有明确目的的访问行为
常见的几类事务 :

- Memory Transaction 内存事务 : 用于访问 Memory Space
- Configuration Transaction 配置事务 : 比如PC开机枚举 FPGA时 通过配置事务完成
- I/O Transaction IO事务: 兼容传统PCI体系保留下来的 IO Space
- Message Transaction 消息事务 : 传递某些控制信息 例如`Interrupt power management`

