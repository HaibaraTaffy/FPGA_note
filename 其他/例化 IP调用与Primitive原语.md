## 例化
---
**在当前设计中创建一个已经定义好的硬件实例**

一般需要包含模块名 例化名
普通例化 属于RTL设计的抽象层级利用 并不是真正的固定的硬件块 会被综合成若干资源
综合的过程中 会发生 资源推导(Inference) 即根据RTL描述的电路行为 由综合器自动选择合适的FPGA底层资源 大部分的FPGA资源是通过这个方式调用的

基本形式 有些可能还会带参数
```
adder u_adder (
    .a(a),
    .b(b),
    .y(y)
);
```


## Primitive原语
---
**FPGA厂商提供的底层硬件资源接口**

厂商把FPGA内部某种真实硬件资源 以HDL可调用的形式暴露出来
可能存在
```
LUT Primitive
FF Primitive
Carry Primitive
Clock Buffer Primitive
I/O Buffer Primitive
```

Verilog中看起来是
```
BUFG u_bufg (
    .I(clk_in),
    .O(clk_out)
);
```

语法上和普通例化几乎一样 但是本质不同
在综合中则是经过 原语例化(Primitive Instantiation)

## IP核
---
往往是**已经封装好的复杂硬件功能模块**
例如
```
FIFO
PLL
Block RAM Controller
DDR Controller
PCIe
Ethernet MAC
DMA
Video Processing
```
IP内部通常会包含
```
RTL Logic
+
Primitive
+
专用硬核
+
配置参数
```
一般可以通过图形化界面来配置IP 然后得到例化示例 在工程中例化即可

## 总结
---
抽象程度 可移植性 RTL > IP > Primitive
底层控制程度 RTL < IP < Primitive
但是越底层不代表越好!