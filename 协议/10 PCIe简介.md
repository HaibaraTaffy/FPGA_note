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

## Transaction Layer 事务层
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

## Split Transaction
---
分离事务 **请求(Request) 和响应(Completion) 不是必须绑在一起立即完成**
可以先请求 然后过段时间再响应

## Tag
---
由于 `Split Transaction` 所以可能出现 乱序
通过 `req` 中打入 `Tag 标签` 用于将响应和原请求对应
例如发送读取请求
```
Read A -> Tag 01
Read B -> Tag 02
Read C -> Tag 03
```
同时都还没回来
之后返回顺序可能是：
```
B
A
C
```
但是可以读取
`Completion Tag 02` 得到正确的请求-响应对

## Flow Control
---
流控 PCIe 使用
**Credit-Based Flow Control 基于信用的流量控制**
它的核心思想是：
```
接收方告诉发送方
我还有多少Buffer空间
↓
发送方根据Credit决定
还能不能继续发送
```
例如：
```
Receiver : 我还能收8个包
↓
告诉Sender Credit = 8
```
Sender每发一个
```Credit - 1```
没有 Credit 以后
```暂时不能继续发```

目的就是
**防止发送方把接收方 Buffer 撑爆**
这里需要稍微分层理解
```
事务层 需要考虑和使用这些Credit
数据链路层 负责通过DLLP传递Credit更新信息
```
**流控功能横跨两层**

## Data Link Layer 数据链路层
---
**保证 TLP 在当前这一条PCIe Link 上可靠传输**
注意 一条Link *每一条Link都有自己的 Data Link Layer*

主要是给 `TLP` 增加一些保护信息 最重要的是 
`Sequence Number 序列号` 和 `LCRC 链路循环冗余校验`

- Sequence Number : 给TLP包打上序列号 接收端就知道有没有漏包 有没有重复
- LCRC : Link Cyclic Redundancy Check : 检查TLP在传输过程中 有没有发生bit错误 类似校验码功能

## ACK 和 NAK
---
**接收端** 收到TLP后 
如果是正确 则通过 `ACK Acknowledgement 确认` 告知发送方
如果是错误 则通过 `NAK Negative Acknowledgement 否定确认` 告知发送方

具体流程简化:
```
Sender
 │ TLP
 ▼
Receiver
 │ CRC检查
 ▼
正确     错误
→ ACK   → NAK
```
**发送端会保存尚未被确认的TLP 若是收到NAK 会重发**

## DLLP
---
Data Link Layer Packer 数据链路层数据包
**在数据链路层自己使用** 一般传递`ACK NAK Credit` 这些信息

## Physical Layer 物理层
---
**负责把0 1 从一块板传到另一块板**
真正的电信号传输

简要拆成四块
```
Physical Layer
│
├─ 串并转换
│
├─ 编码 / 解码
│
├─ Lane和差分信号
│
└─ 链路训练与维护
```

## 串并转换
---
参考[mynote/FPGA资源与基础/13 DDRIO 和 SERDES]
发送端 **并转串** 接收端 **串转并**

## Lane
---
即通道 一条Lane 包括 
`一组TX差分对 + 一组RX差分对` 四根信号线
PCIe 是 **Full Duplex 全双工**
即 **发送和接收可以同时进行**

## LTSSM
---
Link Training and Status State Machine 链路训练与状态状态机
**专门负责 PCIe 链路从上电到正常工作的状态机**

两块设备刚刚接上时 需要先协商
```
对面有没有设备
支持什么速率
有多少Lane可用
Lane是否正常
链路是否稳定
```
一般流程
```
上电
↓
Detect 搜索
↓ 
Polling 轮询
↓
Configuration 配置
↓
L0 正常工作
```

**PCIe PHY上电以后 会自动完成 链路检测 训练 速率和Lane 协商 成功后才正常传输 TLP**
体现在 `link_up` 信号中

## 一次完整信号传输
---
FPGA DMA -> Host Memory
DMA 想写：
```
Address = 0x10000000
Data = ABCD...
```
首先：
### Transaction Layer
```把它做成Memory Write TLP```
里面写：
```
Address
Length
Data
```
↓
### Data Link Layer
加：
```
Sequence Number
LCRC
```
并负责：
```
ACK / NAK
错误重传
```
↓
### Physical Layer
```
编码
串行化
通过TX差分线
发送出去
```
↓
另一端：
```
RX差分线
↓
恢复数据
↓
检查LCRC
↓
解析TLP
↓
发现
Memory Write
↓
写入Host Memory
```