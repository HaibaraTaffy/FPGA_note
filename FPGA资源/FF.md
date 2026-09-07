Flip-Flop 触发器
FPGA 中的 FF 通常以 **D 触发器（D Flip-Flop）** 为基本形式
也存在传播延迟
通常是时序逻辑块 会推导出FF
**reg 不等于 FF** 当 RTL 描述了边沿触发的时序存储行为时 通常会推导出 FF
工程中常把一组FF称为一个Register(寄存器)
>**RTL 中看起来像“保存数据”的结构，并不意味着综合后一定用普通 FF。**
>例如很长的数组、移位寄存器等，综合器可能改用：
>Distributed RAM
>SRL
  BRAM