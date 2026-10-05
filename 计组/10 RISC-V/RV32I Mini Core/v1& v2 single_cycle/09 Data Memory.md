## 模块职责
---
**Data Memory — 数据存储器**
负责保存程序运行过程中使用的数据。
第一版主要服务：
```lw sw```
与 Instruction Memory的区别：

|存储器|保存内容|访问方式|
|---|---|---|
|Instruction Memory|Instruction|当前只读|
|Data Memory|程序数据|可读、可写|

# 1. `lw`的数据路径
---
例如：
```lw x5, 8(x6)```
ALU首先计算 Effective Address：
`Address=RF[x6]+8`
然后访问 Data Memory：
```
RF[x6]
Immediate 8
    ↓
   ALU
    ↓ Effective Address
Data Memory
    ↓ Memory Read Data
Write-Back MUX
    ↓
Register File
```
最终：
`RF[x5]←Memory[RF[x6]+8]`
控制信号：
```
alu_src    = 1
mem_write  = 0
result_src = 1
reg_write  = 1
```

# 2. `sw`的数据路径
---
例如：
```sw x5, 8(x6)```
需要两路数据：
```
RF[x6] + Immediate → Data Memory Address
RF[x5]             → Data Memory Write Data
```
最终：
`Memory[RF[x6]+8]←RF[x5]`
其中：
```
rs1 → Base Register
rs2 → 待写入Memory的数据
```
`sw`没有 `rd`，也不写 Register File。
控制信号：
```
alu_src    = 1
mem_write  = 1
reg_write  = 0
```

# 3. 组合读取、时钟沿写入
---
第一版 Data Memory采用：
```
组合读取
时钟沿写入
```
## 组合读取
```assign read_data = memory[word_index];```
当 `address`发生变化时：
```
address变化
    ↓
word_index变化
    ↓
read_data变化
```
读取不需要 Clock Edge，也不需要 `read_enable`。
非 `lw`指令即使 Data Memory输出数据，后续 Write-Back MUX也不会选择它。
## 时钟沿写入
---
```
always @(posedge clk) begin
    if (write_enable)
        memory[word_index] <= write_data;
end
```
只有同时满足：
```
write_enable = 1
posedge clk到来
```
才会写入 Data Memory。
这里使用 Non-Blocking Assignment，因为 Data Memory保存程序状态。

# 4. Byte Address与Word Index
---
Data Memory接收 ALU计算出的 Byte Address。
每个 Memory Word为：
```32 bit = 4 Byte```
因此：
`WordIndex=ByteAddress/4`
RTL实现：
```
assign word_index = address[ADDR_WIDTH+1:2];
```
当：
```ADDR_WIDTH = 8```
相当于：
```
word_index = address[9:2];
```
地址对应关系：

|Byte Address|Word Index|
|---|---|
|`0x0000_0000`|0|
|`0x0000_0004`|1|
|`0x0000_0008`|2|
|`0x0000_000C`|3|

第一版假设访问地址位于范围内，并且按照4 Byte对齐。

# 5. 存储容量
---
```
ADDR_WIDTH = 8
DEPTH      = 2^8 = 256
Word宽度   = 32 bit
```
总容量:`256×4 Byte=1024 Byte=1 KiB`
Memory Array：
```
reg [31:0] memory [0:DEPTH-1];
```
其中：
```
[31:0]        → 每个Memory Word宽32 bit
[0:DEPTH-1]   → Memory Word数量
```

# 6. 第一版访问宽度
---
当前只支持完整32-bit Word访问：
```
lw → 读取32 bit
sw → 写入32 bit
```
暂时不支持：
```
lb
lbu
lh
lhu
sb
sh
```
因此暂时不需要：
```
Byte Enable
Halfword选择
Load Sign Extension
```

# 7. 初始化
---
仿真开始时将全部 Data Memory初始化为0：
```
initial begin
    for (i = 0; i < DEPTH; i = i + 1)
        memory[i] = 32'b0;
end
```
这主要用于当前功能仿真，避免未写入位置直接读出 `X`。
后续部署到 FPGA时，需要根据实际 Memory资源和初始化方式重新确认。
# 8. 完整RTL
---
```
module data_memory #(
    parameter integer ADDR_WIDTH = 8
)(
    input  wire        clk,
    input  wire        write_enable,
    input  wire [31:0] address,
    input  wire [31:0] write_data,
    output wire [31:0] read_data
);

localparam integer DEPTH = (1 << ADDR_WIDTH);

reg [31:0] memory [0:DEPTH-1];

wire [ADDR_WIDTH-1:0] word_index;

integer i;

assign word_index = address[ADDR_WIDTH+1:2];

assign read_data = memory[word_index];

always @(posedge clk) begin
    if (write_enable)
        memory[word_index] <= write_data;
end

initial begin
    for (i = 0; i < DEPTH; i = i + 1)
        memory[i] = 32'b0;
end

endmodule
```

# 9. 模块性质
---
Data Memory同时包含：
```
组合读取逻辑
+
状态存储结构
+
时钟沿写入逻辑
```
因此它不同于纯组合逻辑的 ALU，也不同于当前只读的 Instruction Memory。
**读取是组合行为，写入是Clocked State Update**

# 10. 核心结论
---
**ALU Result 提供 Address RF[rs2] 提供 Write Data
lw 读取 Memory 并写回 RF sw 读取 RF 并写入 Memory
Data Memory 组合读 时钟沿写**