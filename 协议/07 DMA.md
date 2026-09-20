>参考来源 : [DMA原理，步骤超细详解，一文看懂DMA-CSDN博客](https://blog.csdn.net/2401_87198107/article/details/142341269) 只参考了一部分
>GPT回答 更多的参考来源

## 基本定义
---
DMA 全称 Direct Memory Access 直接存储器访问
**DMA传输 让专门的DMA Controller 或 DMA Engine 直接参与数据搬运**

主要是为了解决 大量数据的转移 过度消耗CPU资源
的问题 使得CPU可以节省资源
**但并不是完全不占用资源 DMA也会占用一定的带宽**

## 传输方式
---
主要涉及四种情况的数据传输 :
- 外设到内存
- 内存到外设
- 内存到内存
- 外设到外设
但本质上一样 都是从**一个数据端点读取数据 然后写入另一个数据端点**

## DMA传输参数
---
核心参数 :
1. 数据原地址
2. 数据目标地址
3. 数据传输量
4. 传输模式 即进行多少次传输

一般 配置前三个 DMA控制器就会启动数据传输
当剩余传输数据量为0时 达到传输终点
也可以设置 `循环传输模式` 只要剩余传输数据量不是0 且DMA处于启动状态 就会发生数据传输

## Stream 和 Memory 之间的桥梁
---
先介绍 FPGA中常见的两个接口

>Memory-Mapped 内存映射接口

数据通过地址访问 通过 `Address` `Data` `Write_Enable` 等信号进行数据传输与控制

> Stream Interface 流接口

一般没有内存地址概念 只有 `data` `valid` `ready` 数据不断向前流动

很多FPGA DMA的一个重要职责 
```
Stream
   ↓
DMA
   ↓
Memory-Mapped
   ↓
DDR
```

## AXI DMA
---
典型的AXI DMA结构
```
                AXI DMA

AXI Stream ──────────────> AXI Memory Mapped
                                │
                                ▼
                               DDR
```

通常分为两个方向

> S2MM 流到内存映射

```
FPGA数据流
   ↓
AXI Stream
   ↓
DMA
   ↓
DDR
```

> MM2S 内存映射到流

```
DDR
 ↓
DMA
 ↓
AXI Stream
 ↓
FPGA逻辑
```

## 常见配置参数
---
```
Source Address 源地址
Destination Address 目的地址
Transfer Length 传输长度 一次搬多少数据
Transfer Width 传输位宽 一次搬多少位
Source Address Increment 源地址递增
Destination Address Increment 目的地址递增
Burst Length 突发长度
Transfer Mode 传输模式
Interrupt Enable 中断使能
```

## 地址递增
---
在 Memory -> Memory 时 一般需要进行地址递增 例如
```
src = 0x1000
dst = 0x2000
第一次
0x1000 -> 0x2000
第二次
0x1004 -> 0x2004
...
此时
Source Increment = Enable
Destination Increment = Enable
```

但是如果 UART -> Memory 则不同
UART数据寄存器可能永远是
`0x40001000` 无需递增
因此
```
Source Increment = Disable
Destination Increment = Enable
```

## 工作流程
---
```
① CPU 配置 DMA
        │
        ▼
② DMA 获得任务
        │
        ▼
③ DMA 请求总线
        │
        ▼
④ DMA 从 Source 读取数据
        │
        ▼
⑤ DMA 写入 Destination
        │
        ▼
⑥ 更新地址
        │
        ▼
⑦ Length -= transferred_size
        │
        ▼
   Length == 0 ?
      │      │
     No     Yes
      │      │
      └──────┤
             ▼
        Transfer Done
             │
             ▼
        Interrupt CPU
```

## Bus Master
---
总线主设备 **能主动发起总线事务的设备**
例如 `CPU DMA GPU NPU PCIe` 都可能

例如 现代DMA的使用
```
CPU
 │
 │ 配置 DMA
 ▼
DMA

随后

DMA ──────> AXI Interconnect ──────> DDR
```

## Arbitration Burst Beat
---
- Abitration 仲裁 多个Master时 会出现总线占用冲突 这个时候就需要仲裁
系统需要决定 访问顺序
影响 `Bandwidth 带宽` `Latency 延迟`

- Burst 突发传输 攒够数据 一次传输 发送多次数据
例如
```
传输 Address
   ↓
传输 Data
传输 Data
传输 Data
传输 Data
传输 Data
传输 Data
……
```
可以理解为一个阀门 Address一有效 就开始不断传输Data 效率很高

- Beat 单次总线数据传输单元 基本和数据宽度对应
**一个 Burst 由 多个 Beat 组成**

## Simple Scatter-Gather 和 Circular DMA
---
Simple DMA 可以理解为 **一次配置 一次搬运**
抽象的理解为 CPU配置 `搬这一块` DMA搬完 `Done`
一般配置 `Source Destination Length`

更高级的是 SG DMA 分散聚集DMA
CPU提前 准备很多 `Descriptor 描述符` 
例如
```
Descriptor 0

Source = A
Destination = B
Length = 1024
Next = Descriptor 1
```
DMA 自己不断读取 Descriptor 减少CPU干预
```
Descriptor0
↓
搬数据
↓
Descriptor1
↓
搬数据
↓
Descriptor2
↓
……
```
常见高速 `Network PCIe Video Storage`

还有一种 Circular DMA 循环DMA

例如开两个 Buffer
```Buffer A Buffer B```
形成
```
DMA
 ↓
Buffer A
 ↓
Buffer B
 ↓
Buffer A
 ↓
Buffer B
……
```
CPU 可以
```
DMA写A
CPU处理B

DMA写B
CPU处理A
```
这就是很常见的
**Ping-Pong Buffer 乒乓缓冲**

## DMA Controller
---
**DMA是一种数据传输机制
DMA Controller 是实现 DMA 机制的具体硬件模块**

```
CPU
 │
 │ 配置
 ▼
DMA Controller
 │
 ├─ Control Interface
 │    接收 CPU 配置
 │
 ├─ DMA Engine
 │    真正执行读 写 地址更新 长度计数
 │
 └─ Data Interface
      访问 Memory / Peripheral
```

## 最简单DMA RTL设计
---
```Verilog
module simple_dma (
    input  wire        clk,
    input  wire        rst_n,
    // CPU / Control
    input  wire        start,
    input  wire [31:0] cfg_src_addr,
    input  wire [31:0] cfg_dst_addr,
    input  wire [31:0] cfg_length,
    output reg         busy,
    output reg         done,

    // Memory Read Interface
    output reg         rd_req,
    output reg  [31:0] rd_addr,
    input  wire        rd_ready,
    input  wire        rd_data_valid,
    input  wire [31:0] rd_data,

    // Memory Write Interface
    output reg         wr_req,
    output reg  [31:0] wr_addr,
    output reg  [31:0] wr_data,
    input  wire        wr_ready
);

    // DMA Internal Registers
    reg [31:0] src_addr_reg;
    reg [31:0] dst_addr_reg;
    reg [31:0] remain_reg;
    reg [31:0] data_reg;

    // FSM
    localparam IDLE       = 3'd0;
    localparam READ_REQ   = 3'd1;
    localparam READ_WAIT  = 3'd2;
    localparam WRITE_REQ  = 3'd3;
    localparam DONE       = 3'd4;
    reg [2:0] state;

    always @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            state        <= IDLE;
            src_addr_reg <= 32'd0;
            dst_addr_reg <= 32'd0;
            remain_reg   <= 32'd0;
            data_reg     <= 32'd0;
            rd_req       <= 1'b0;
            rd_addr      <= 32'd0;
            wr_req       <= 1'b0;
            wr_addr      <= 32'd0;
            wr_data      <= 32'd0;
            busy         <= 1'b0;
            done         <= 1'b0;
        end
        else begin
            done <= 1'b0;
            case (state)
                // IDLE 读取配置 拉高忙 进入请求阶段
                IDLE: begin
                    rd_req <= 1'b0;
                    wr_req <= 1'b0;
                    busy   <= 1'b0;
                    if (start) begin
                        src_addr_reg <= cfg_src_addr;
                        dst_addr_reg <= cfg_dst_addr;
                        remain_reg   <= cfg_length;//配置传输长度
                        busy         <= 1'b1;
                        state        <= READ_REQ;
                    end
                end
                // READ REQUEST 发送请求 并且驱动rd_addr
                READ_REQ: begin
                    rd_req  <= 1'b1;
                    rd_addr <= src_addr_reg;
                    if (rd_ready) begin//上游准备好后 把请求拉低 进入读等待
                        rd_req <= 1'b0;
                        state <= READ_WAIT;
                    end
                end
                // WAIT READ DATA
                READ_WAIT: begin //检验现在是否可以读
                    if (rd_data_valid) begin
                        data_reg <= rd_data;
                        state <= WRITE_REQ;//已经读出 开始请求写
                    end
                end
                // WRITE 拉高写请求 配置写地址 写数据
                WRITE_REQ: begin
                    wr_req  <= 1'b1;
                    wr_addr <= dst_addr_reg;
                    wr_data <= data_reg;
                    if (wr_ready) begin
                        wr_req <= 1'b0;//下游准备好 拉低请求
                        if (remain_reg <= 32'd4) begin
                            remain_reg <= 32'd0;//如果 remain_reg 减到4了 
					                            //就进入Done
                            state <= DONE;
                        end
						//没传完 就继续传
                        else begin
							//一次传四个字节嘛 32位 所以4个4个单位变化
                            src_addr_reg <= src_addr_reg + 32'd4;
                            dst_addr_reg <= dst_addr_reg + 32'd4;
                            remain_reg <= remain_reg - 32'd4;
                            state <= READ_REQ;
                        end
                    end
                end
                // DONE 完成一个DMA Transfer
                DONE: begin
                    busy <= 1'b0;
                    done <= 1'b1;
                    state <= IDLE;
                end
                default: begin
                    state <= IDLE;
                end
            endcase
        end
    end
endmodule
```

一般不会自己写 DMA Controller 基本可以调用IP

可以发现 其实使用了握手协议(Handshake Protocol)
但是DMA并不等于 握手协议

| 概念          | DMA        | 握手协议        |
| ----------- | ---------- | ----------- |
| 本质          | 数据搬运机制     | 通信协调规则      |
| 关注什么        | 从哪搬到哪 搬多少  | 这一拍能不能传     |
| 是否需要地址      | 通常需要       | 不一定         |
| 是否需要长度      | 通常需要       | 不需要         |
| 是否有任务结束     | 有          | 通常没有        |
| 是否可以包含 FSM  | 可以         | 可以          |
| 是否可以独立成为大模块 | 可以         | 通常只是接口的一部分  |
| 两者关系        | DMA 经常使用握手 | 握手可以服务于 DMA |
