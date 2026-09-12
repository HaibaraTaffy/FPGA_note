先区分 
BRAM 属于 On-Chip Memory 片上存储 相对较小 总容量往往只有几Mbit到几十Mbit 只适合小缓存
DDR 属于 Off-Chip Memory 片外存储 相对较大 可以存储较大的数据 **需要使用IO口来连接**

## RAM SRAM DRAM SDRAM
---
- RAM : 随机存储器 参考数电中的定义
- SRAM : 静态随机存储器
	- 正常供电 数据可以保持 不需要周期刷新
	- 特点 : 快 控制相对简单 不需要Refresh 单bit占用面积较大 成本较高 容量密度较低
	- FPGA中很多片上RAM资源的存储单元属于SRAM类实现
- DRAM : 动态随机存储器
	- 利用电容中的电荷状态保存bit 需要周期性Refresh
	- 特点 : 密度高 容量大 单位容量成本低
- **SDRAM : 同步动态随机存储器 S意为同步 DRAM的操作与时钟同步**

>DDR : 全称DDR SDRAM DDR指的是 Double Data Rate 一个时钟周期内 在两个时钟边沿均传输数据

## DDR频率
---
DDR频率是由 MT/s来衡量的 即每秒百万次传输
由于DDR每个时钟周期进行两次传输 于是存在关系
`Data Rate = 2 x Clock Rate`
例如 DDR4-3200
理解为 3200MT/s 时钟周期为 1600MHz

## DDR数据位宽
---
DDR芯片具有数据总线 具有数据位宽 通常 8 16 64bit
例如64bit=8Byte 3200MT/s
则理论的峰值带宽 : 3200 x 10^6 x 8 B即25.6GB/s
得出公式 `Bandwidth = Transfer Rate x Bus Width`

## DDR内部组织
---
具有Bank(块) Row(行) Column(列)
![[FPGA_DDR_结构与接口.svg]]
一个逻辑地址 会被控制器转换成 Bank+Row+Column

## DDR读写简化流程
---
先打开一整行 Activate(激活) Row 然后再这一行中访问(读或写)一列 Column
① ACTIVATE Row
       ↓
② Row 被打开
       ↓
③ READ / WRITE Column
       ↓
④ PRECHARGE

>感觉和计组的存储体一样

## Row Buffer
---
当某一Row被Activate后 该Row的内容会进入内部的 `Row Buffer(行缓存)`
> 对应示意见本节首图的“行缓冲访问”“FPGA 到 DDR 的分层”和“数据复用路径”。

ROW Hit : 访问Activate Row的连续Column 效率很好
但若是下一次访问其他的Row 就得重新打开Row

于是 **连续访问通常比零散随机访问效率高**

## Burst
---
因此 DDR喜欢 `Burst Transfer` 即突发传输
一次请求连续传输一串数据 (由Row Buffer可以很好理解)

## DDRController + DDR PHY
---
> 对应示意见本节首图的“行缓冲访问”“FPGA 到 DDR 的分层”和“数据复用路径”。

Controller 注重逻辑 负责
```
地址映射
读写调度
Bank / Row 管理
Refresh
Command Scheduling命令调度
```

PHY 注重物理层面 负责
```
真正的高速电气接口
DQ
DQS
Clock
采样
延迟校准
高速I/O
```

但是这些一般都封装成IP核 一般IP核中都包含
```
Controller
+
PHY
+
Calibration Logic 校准逻辑
+
底层 I/O Primitive
```
然后 **用户侧能看到的一般是 Memory Interface(内存接口)/AXI总线**

## 区分BRAM和DDR
---
```
BRAM
容量小
延迟低
片上
访问简单

DDR
容量大
高连续带宽
片外
访问复杂
```


## BRAM和DDR联合应用
---
一个Data Reuse(数据复用)和Memory Hierarchy(存储层次)的雏形
```
> 对应示意见本节首图的“行缓冲访问”“FPGA 到 DDR 的分层”和“数据复用路径”。
```

利用DDR的大容量来存储大量数据 通过提前搬运到BRAM中 使得读取操作变快捷 (有点像内存 外存的感觉?)