即输入输出接口 本笔记只探讨FPGA部分的IO 
>参考资料 : [FPGA的基本架构、IO命名方式和作用_fpga芯片引脚定义查询-CSDN博客](https://blog.csdn.net/u011995183/article/details/122781967)
>GPT回答

## 概况
---
从PCB到FPGA内部
```
外部器件 / PCB
      │
      │ 电压、电流、差分信号……
      ▼
Physical Pin / Ball
      │
      │ A13、B7、C12……
      ▼
Pin 的功能属性
      │
      │ User I/O / P / N / Clock-capable...
      ▼
I/O Bank
      │
      │ VCCO / I/O Standard
      ▼
I/O Block
      │
      │ IBUF / OBUF / IOBUF
      │ Register / Delay / DDR / SERDES...
      ▼
FPGA Internal Signal
      │
      ▼
LUT / FF / BRAM / DSP / Routing
```

## Pin
---
FPGA芯片与PCB建立物理连接 封装上有大量的 `Pin/Ball` 即引脚/管脚
![[Pasted image 20260828184506.png]]
类似这样 为了便于管脚绑定 于是存在物理命名

>物理命名规则

FPGA 通常具有大量封装引脚（Pin/Ball） 其中一部分为用户 I/O 此外还包括电源 地 配置及其他专用功能引脚 BGA 等封装通常使**字母+数字**标识物理 Ball 位置 例如 A13  如图 圈起来的IO口的物理命名则是`A13` 
一般来说都是这么命名

## I/O口
---
每个管脚都会有自己的功能 于是存在第二套命名规则 即功能命名

>功能命名规则

以Xilinx的命名为例 
**差分IO**格式一般为 : **IO_LXXY#/IO_XX** 详细展开:
- IO 代表用户 IO；
- L 代表差分，XX 代表在当前 BANK 下的唯一标识号，Y处是P或者N 表示 LVDS 信号的 P 或者 N；
- `#`代表Bank 号。
比如，我们的原理图中有一个 IO 的名字为：IO_L13P_T2_MRCC_12，那通过功能命名的规则我们就可以知道，这是一个用户 IO，支持差分信号，是 BANK12 的第 13 对差分的 P 端口，与此同时它也是全局时钟网络输入管脚（MRCC 是全局时钟网络）。

**单端IO**格式一般为 : IO_X_XX
- 第一个X代表单端IO的编号
- 后两个#代表单端IO所在的Bank。
举例：IO_25_12，代表第25个单端IO，IO位于第12BANK。

功能命名用于描述 **物理Pin在FPGA芯片内部具有什么功能属性**

## User I/O
---
如果某个Pin 是用户IO 则意味着可以被用户逻辑配置为
```
Input 输入端口
Output 输出端口
Bidirectional (INOUT)
```

## 单端信号 Single-Ended
---
最常见的GPIO 使用`一根信号线 + 共同参考电位(一般是GND)` 来判断逻辑状态 这里的逻辑0/1 需要通过真实的电压范围来表示

## I/O Standard
---
规定了 Pin侧应该遵守怎样的电气接口规则 即接口标准
我们常见的
```
LVCMOS33
LVCMOS18
LVTTL
LVDS
...
```
就属于IO Standard
包括可能涉及
```
供电电压
输入阈值
输出电平
驱动能力
差分/单端方式
终端相关要求
```

这些规则共同约束了 Pin上的电气行为 由此来界定RTL里的0/1

## BANK
---
为了便于管理和适应多种电器标准 以及供电和电气资源不能为每个Pin完全独立配置 FPGA 的IO部分被划分为若干个组
即**Bank**

Bank是一组物理位置和特性相近的IO的总称
- 每个 Bank 的接口标准由其接口电压 VCCIO 决定 限制该Bank能使用哪些接口标准 但一个VCCIO不一定只对应唯一一种接口标准
- 一个 Bank 只能有一种 VCCIO
- 但不同 Bank 的 VCCIO 可以不同
**VCCIO** : I/O Bank的输出侧供电电压之一

只有相同电气标准和物理特性的端口才能连接在一起 VCCIO 电压相同是接口标准的基本条件 同一Bank的电压的基准是一致的
因此 通常如果我们需要各种不同标准的电压 可以通过给到Bank的电压基准不同的方式来实现多种电平标准的输入输出 通常封装越大 Bank数量也越多 可以支持电压标准也越多

## 差分信号 Differential
---
利用两根线 `P N` 来共同表示一个信号 接收端主要是观察`V_P - V_N` 来判断逻辑状态
一般来说 V_P - V_N > 0 则表示逻辑1 反之表示逻辑0
**差分信号 可以抑制一部分共模噪声(当外界对两条信号都有干扰时 相减后干扰会被抑制)**
另外差分也常适合：
- 高速信号
- 较低摆幅
- 时钟
- 高速串行链路
## I/O Block
---
**IO Block 负责Pin电气世界与FPGA内部逻辑世界之间的接口**

对于单端输入 从Pin进来的信号会先进入`Input Buffer` 再进入内部信号
对于单端输出 从内部出来的信号会先进入`Output Buffer`再输出到物理管脚

对于差分输入信号 差分PN则会通过`Differential Input Buffer` 转换成内部的一个数字逻辑信号
对于差分输出信号 内部的一个数字逻辑信号 通过`Differential Output Buffer` 转换成外部的一对差分PN

## I/O Block 的 Primitive
---
单端情况 :
- 输入 Pin -> FPGA `IBUF`(Input Buffer)
- 输出 FPGA -> Pin `OBUF`
- 双向 FPGA <-> Pin `IOBUF`

差分情况 :
- 输入 Pin -> FPGA `IBUFDS`(Input Buffer Differential s表复数)
- 输出 FPGA -> Pin `OBUFDS`
- 双向 FPGA <-> Pin `IOBUFDS`

INOUT通过三态门进行切换

## 现代I/O Block
---
现代的IO Block更加复杂
```
                    I/O Block

Pin
 │
 ├── Input Buffer
 │
 ├── Output Buffer
 │
 ├── Tri-State Control
 │
 ├── Input Register
 │
 ├── Output Register
 │
 ├── Delay Element
 │
 ├── DDR I/O
 │
 └── SERDES related resources
```

很多时序资源直接放在IO边缘附近 能够尽可能避免高速IO的**不可控的Routing Delay**
FPGA IO本身就是一套**专门的边界硬件架构**