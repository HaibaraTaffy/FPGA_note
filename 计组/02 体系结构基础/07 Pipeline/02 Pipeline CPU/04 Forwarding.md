即数据前递 解决 
```
某个 Result 已经产生 但还没有到达正常存储位置 后续 Hardware 又立即需要这个 Result 
能否建立更短的数据路径 直接把 Result 送给 Consumer
```
## Forwarding 本质
---
Forwarding 增加旁路
```
EX/MEM Result ───────────┐
                        │
MEM/WB Result ───────────┼→ Forwarding MUX → ALU Input
                        │
ID/EX Register Data ─────┘
```
本质就是
**检测 Operand 是否已经过期 如果过期 寻找 Pipeline 后级中更新的数据 通过 MUX 选择最新的 Result**
只是改变当前 EX Stage 实际使用的数据来源

## Producer 和 Consumer
---
Producer 数据生产者 产生某个 Register 新值的 Instruction
Consumer 数据消费者 读取这个 Register 的后续 Instruction
**都是描述 Instruction 的名词**

于是 Forwarding Logic 需要回答 :
```
当前 EX Stage 的 Consumer 所需要的 rs1 和 rs2 
是否由 Pipeline 后级中的 Producer 产生
```

## Forwarding Unit
---
