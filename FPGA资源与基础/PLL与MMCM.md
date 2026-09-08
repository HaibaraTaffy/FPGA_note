## PLL
---
即**锁相环** (Phase-Locked Loop)
**通过反馈 让内部产生的时钟与参考时钟保持确定的频率和相位关系**
可以进行时钟频率的合成 产生新的时钟频率
并不是普通LUT逻辑实现的数字分频器 而是**专用的时钟资源**

高度简化版本
```
Reference Clock
      │
      ▼
┌──────────────┐
│Phase/Frequency│
│   Detector    │
│(相位/频率检测器)│
└──────┬───────┘
       │
       ▼
   Control
       │
       ▼
┌──────────────┐
│  Oscillator  │
│	(振荡器VCO) │
└──────┬───────┘
       │
       ├────────────► Output Clock
       │
       └──── Feedback(反馈) ────┐
	                            │
                                └──► Detector
```
通过不断比较`Reference Clock 和 Feedback Clock` 调整`Feedback Clock`
如果反馈 Clock：
- 太快 → 调慢
- 太慢 → 调快
- 相位提前 → 调整
- 相位落后 → 调整
直到二者达到目标关系 这就是所谓的`Lock`


## Phase
---
就是相位 参考数电知识
Phase Difference : 相位差
Phase Shift : 相位偏移

## 分频 倍频
---
- N分频 : `f_out = f_in / n`
- N倍频 : `f_out = f_in * n`


PLL可以在反馈路径中 加入倍频(M)/分频(D)关系
在输出端也可以进行分频(O) 所以有公式
`f_out = f_in X M /(D X O)`
具体的M D O 允许选择哪些参数 需要看具体的FPGA的PLL/MMCM规格

PLL/MMCM可以做更多时钟上的操作
```
Frequency Multiplication
Frequency Division
Phase Shift
Clock Conditioning
Multiple Clock Outputs
```

PLL一般支持输出多个频率


## MMCM
---
即Mixed-Mode Clock Manager 混合模式的时钟管理器
它可以提供：
- 倍频
- 分频
- 多路 Clock 输出
- Phase Shift
- Duty Cycle 调整
- Clock Conditioning

## LOCKED信号
---
LOCKED信号 在PLL建立起稳定状态后 置1 用于向外部逻辑表示内部状态 可以作为一些逻辑的复位 使其在得到稳定的时钟之后才开始工作
本意表示 输出Clock达到了所要求的频率/相位关系

## Jitter
---
Clock不可能理想稳定 会产生 Clock Edge相对于理想时间位置的短期变化 称为 Jitter
>区分Skew Jitter 
>Skew是 同一个Clock边沿到达不同位置的时间差
>Jitter 是Clock边沿自身相对于理想时间位置的波动

PLL有助于改善Clock 进行一定的Clock Conditioning(时钟调节)

## IP核配置
---
一般都是通过IP核来配置



