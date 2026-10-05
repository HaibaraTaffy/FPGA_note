## 模块职责
---
Instruction Memory根据 PC提供的 Instruction Address，输出对应的32-bit Instruction。
```
PC
 ↓ Instruction Address
Instruction Memory
 ↓
32-bit Instruction
```
第一版 Instruction Memory在 CPU运行期间只读，不提供写入端口。

## Byte Address与Word Index
---
RISC-V按照 Byte编址，一条基础 RV32I Instruction占4 Byte。
```
PC = 0x0000_0000 → memory[0]
PC = 0x0000_0004 → memory[1]
PC = 0x0000_0008 → memory[2]
PC = 0x0000_000C → memory[3]
```
转换关系：
`WordIndex = ByteAddress / 4`
RTL中相当于去掉地址最低两位：
```
assign word_index = address[ADDR_WIDTH+1:2];
```
当 `ADDR_WIDTH=8`时：
```
word_index = address[9:2];
```

## 存储容量
---
```
ADDR_WIDTH = 8
Word数量   = 2^8 = 256
Word宽度   = 32 bit = 4 Byte
总容量     = 256 × 4 Byte = 1 KiB
```
1 KiB空间的 Byte Address范围为：
```
0x0000_0000～0x0000_03FF
```
最后一个4 Byte对齐的 Instruction Address为：
```
0x0000_03FC
```
第一版假设输入地址始终位于范围内并正确对齐。

## 组合读取
---
Instruction Memory采用组合读取：
```
address变化
    ↓
word_index变化
    ↓
instruction变化
```
不需要 Clock Edge。
```
assign instruction = memory[word_index];
```
这是当前教学用 Single-Cycle CPU的功能模型。后续部署到 FPGA时，再处理 BRAM同步读取 Latency。

## NOP初始化
---
NOP — No Operation — 空操作。
RISC-V常用：
```
addi x0, x0, 0
```
作为 NOP，对应机器编码：
```
00000013
```
初始化时先将所有 Memory Word填充为 NOP：
```
for (i = 0; i < DEPTH; i = i + 1)
    memory[i] = 32'h0000_0013;
```
然后使用：
```
$readmemh(MEM_FILE, memory);
```
加载实际程序。
这样没有被程序文件覆盖的位置仍然保存 NOP。

## Hex程序文件
---
```
programs/imem_test.hex
```
内容：
```
00500093
00700113
002081B3
40208233
```
对应：
```
addi x1, x0, 5
addi x2, x0, 7
add  x3, x1, x2
sub  x4, x1, x2
```
Hex文件每行对应一个32-bit Memory Word。

## 核心结论
---
**PC提供Byte Address，Instruction Memory使用Word Index读取Instruction**