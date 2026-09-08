时序分析 关键在于
**数据能不能在下一个采样时刻之前正确到达**

## FF2FF
---
直接看一个最简单的 
FF -> 组合逻辑 -> FF
```
          Data Path

FF_A ──► LUT ──► Carry ──► Routing ──► FF_B
 ▲                                         ▲
 │                                         │
 └──────────── Clock Network ──────────────┘
```
FF_A 又称 `Launch FF` FF_B 又称 `Capture FF`

设f_clk = 100Mhz 则T_clk = 10ns
数据大约有一个T_clk的时间 可以从FF_A 跑到 FF_B
但是中间还有各种延迟!!!

- t_CQ : Clock_to_Q Delay
时钟边沿到了 需要等一个t_CQ Q才能从FF_A里输出

- t_logic : 在硬件中传播的延迟 与Logic Depth有关

- t_route : 数据在物理连线上移动 产生的延迟
  >实际FPGA中 Routing Delay往往非常重要 甚至可能比Logic Delay更突出
  
  - t_setup : 数据到达Captrue FF后 也不能踩着Clock Edge到 要求 **数据必须在采样Clock Edge之前提前稳定一小段时间 即建立时间**

综上 我们可以得到一个公式
`t_CQ + t_logic + t_route + t_setup = Data Path Delay <= T_clk`
这是一个最基本的 暂时忽略Skew 和 Uncertianty

加入了Skew 和 Uncertainty
`Data Path Delay <= 实际可用的Clock Timing Budget` 
## Slack
---
即裕量
`Slack = Required Time - Arrival Time`
当 Slack > 0 (Positive Slack) 则说明 **Timing Met(满足时序)**
当 Slack < 0 (Negative Slack) 则说明 **Timing Violation(时序违例)**

## Clock Frequency 与 Timing
---
由上两小节很容易可以想到 f_clk 提高 则T_clk减小 物理逻辑没有因为Clock Frequency提高而提高 于是Slack会变得紧张
**提高Clock Frequency 本质上是在压缩允许的数据传播时间**

虽然降低Clock Frequency 可以解决一定的时序问题 但一般是通过增加`Setup Budget 建立预算` 来解决
对于Hold问题 即数据在当前采样边沿之后变化得太快 不太能优化
于是 **单纯降低Clock Frequency 通常不能直接解决Hold Violation**

## Critical Path
---
关键路径 可以理解为
**当前Timing要求下最难满足 最限制系统频率的路径之一**
于是说时序优化 一般 先从`Worst Timing Paths` 入手

## Pipeline 与 Timing
---
很容易理解 通过在长组合路径中插入FF 构成Pipeline 通过缩短单极`Critical Path(关键路径)`来满足时序裕量
通常是用Latency 来换取 Frequency/Throughout 能力

## Clock Uncertainty 
---
由于Clock并非完美 且还有其他的干扰存在 所以在Timing分析时 留出一定的`Clock Uncertainty`

## Hold Time
---
即保持时间
在Clock Edge到来之后 也要求输入继续保持稳定一小段时间 即`Hold Time`

## Max/Min Delay 
---
对于Setup 担心 数据路径太慢 关注 `Maximum Delay`
对于Hold 担心 数据路径太快 关注 `Minimum Delay`

## Timing Closure
---
即**时序收敛**
指的是 设计在目标约束下经过综合 布局布线和必要优化后 关键 Timing 要求都被满足 

于是整个典型流程
```
RTL
 ↓
Synthesis
 ↓
Implementation
 │
 ├─ Placement
 └─ Routing
 ↓
Static Timing Analysis
 ↓
Timing Met？
 ├─ Yes → Closure
 │
 └─ No
     ↓
分析Critical Path
     ↓
修改RTL / Pipeline / Placement / Constraints...
     ↓
重新Implementation
```