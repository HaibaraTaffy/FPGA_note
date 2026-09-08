即 Block RAM 是FPGA芯片内部预先制造好的**专用片上RAM硬件资源**
属于一种专用资源 存储资源

通常的BRAM Block大小 18Kbit 36Kbit
36Kbit = 36 x **1024** bit ≈ 4.5KB

## 基本接口
---
![[FPGA_BRAM_接口.svg]]

|信号|作用|
|---|---|
|Address|指定访问哪个存储位置|
|Data In|写入的数据|
|Data Out|读取的数据|
|WE|Write Enable，写使能|
|CLK|时钟|
|EN|有些架构还有整体使能|

## Width(位宽) Depth(深度)
---
存储器的容量通常写成 : Depth x Width
例 : 1024 x 32bit 1024个地址 每个地址32bit 总容量 1024 x 32 = 32768bit
地址位宽 log2(Depth) 以上面为例 地址位宽为 10bit

在总容量固定的情况下 可以在一定范围内调整“宽度”和“深度”的组织方式
例如 4096 x 9 2048 x 18

## Single-Port RAM
---
单端口RAM 包含
```
Address
Data In
Data Out
WE
CLK
```
虽然有Din Dout 但是共用地址共用端口 
在一个周期内只能对 **一个地址 做读或写操作**

## Dual-Port RAM
---
双口RAM 两个端口可以访问同一块存储空间
> 上图左侧为单端口接口，右侧为双端口接口示意。

## Simple Dual Port
---
简单双端口 一个端口写 一个端口读 
存在addr_in din we_in clk_in 这是a端口 只写
也存在 addr_out dout clk_out 这是b端口 只读
在一个周期内 支持往addr_in中写 从addr_out中读


## True Dual Port
---
两个端口都能读写

## 读写独立时钟
---
部分FPGA BRAM支持 两个不同端口受不同的独立clk控制 可以处理跨时钟域(CDC)结构

>但是 有两个独立时钟并不意味着可以随意把BRAM当成CDC工具使用 尤其两个端口访问同一地址时存在具体的读写冲突规则

## 读延迟
---
很多FPGA BRAM 使用**同步读取**
存在着读延迟 具体时序取决于BRAM配置 以及是否启用额外输出寄存器

区分同步读和异步读 : **读取过程是否依赖时钟沿?**
- 异步读 (Asynchronous Read)
  地址一变 输出也变 **不依赖时钟沿** 但不意味着无延迟 还是存在物理延迟的
``` Verilog
assign dout = mem[addr];
```

- 同步读 (Synchronous Read) 
  地址变化 要等到时钟沿 输出才变化 **依赖时钟沿** 即必须经过有效时钟事件 读取结果才更新
``` Verilog
always @(posedge clk) begin
    dout <= mem[addr];
end
```

读延迟的计算 :
1. 确定同步还是异步
2. 接口Latency确定是多少cycle 大部分可以在IP配置时确定
3. 真正timing时看ns
## 推断BRAM
---
RTL 写法必须符合综合器支持的 BRAM 推导模板 才能稳定推导成 BRAM
但也可以通过IP来进行配置 这种更加稳定方便
最后还支持原语例化

## 拼接扩展
---
如果遇到一组数据 一个BRAM放不下 综合器/IP会把多个物理BRAM拼起来
可能的形式 :
- 深度级联
- 宽度并联
- 两者组合
在RTL中看到的是一个RAM 实际实现的时候 底层可能使用多个BRAM Block