DDR IO和SERDES都属于**FPGA边缘高速IO资源**
## DDRIO
---
即 `Double Data Rate IO` 一个Clock周期内 在上升沿和下降沿都传输数据
也就是 `2 transfers per clock cycle`
请注意区分 DDR 只是一个行为方式 DDR SDRAM和DDR IO都采用了这个方式

输出时
FPGA IO区域有专门的 `DDR Register/DDR IO Primitive` 来进行DDR输出 这些资源一般比较靠近IO Block

输入时 也有`DDR Input Register` 来进行采集 举例
一个外部串行/高速信号
```
A B C D E F ...
```
可以被拆成：
```
rise_data：A C E ...
fall_data：B D F ...
```
然后交给 FPGA 内部逻辑

## SERDES(SER DES)
---
即 `SERializer/DESerializer 串行器/解串器`

解决 一个Pin每周期只能传少量的bit 但内部逻辑想要一次处理多个bit 的问题

>Serializer 并行转串行 Parallel -> Serial

使用Serializer 可以将并行数据转成串行数据按时间顺序发送
```
8-bit Parallel Data
        │
        ▼
   Serializer
        │
        ▼
      1 Pin
        │
        ▼
D7 D6 D5 D4 D3 D2 D1 D0
按时间顺序发送
```

>Deserializer 串行转并行 Serial -> Parallel

在接收端则反之 一次性收集多个bit 然后再统一送到FPGA中
```
1 Pin
 │
 ▼
D7 D6 D5 D4 D3 D2 D1 D0
 │
 ▼
Deserializer
 │
 ▼
8-bit Parallel Data
```

因为Pin的个数一定 该资源很宝贵 所以利用`SERDES` 可以用更少的Pin传更高的数据率 本质上是在 **Pin数量和单Pin的速率之间做权衡**

当然 SERDES也是一种专用资源 FPGA会提供`Dedicated IO SERDES Resource`来处理

## 二者关系
---
SERDES和DDR不是互斥的 可以同时使用
例如一个 Serializer 可能：
```
内部并行数据
    │
    ▼
SERDES
    │
    ▼
DDR Output
    │
    ▼
Pin
```
这样每个 Clock 两个边沿都可以发送 bit。

## Source-Synchronous(源同步)
---
在高速IO中 我们会用源同步的方式 
在发送端 不仅发Data 还一起发一个相关的Clock/Strobe
在接收端 可以用这个伴随Clock去采集Data
(可以想起OV5640的Pclk)
数据和采样参考一起传 可以更好地控制二者之间的相对 Timing

## Delay Element
---
在高速输入时 可能需要将采样点稍微前后移动 可以使用专用的`Delay Element` 很多FPGA IO Blcok中会有 Programmable Delay