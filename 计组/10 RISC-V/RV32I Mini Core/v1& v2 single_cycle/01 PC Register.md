## PC的职责
---
PC — Program Counter — 程序计数器。
PC Register负责保存当前指令地址：
```
pc = Current PC
```
在 Clock Edge 到来时：
```
rst = 1 → pc更新为RESET_VECTOR
rst = 0 → pc更新为next_pc
```
两个 Clock Edge 之间，PC保持原来的状态。
## RTL结构
---
```Verilog
module pc_reg #(
    parameter [31:0] RESET_VECTOR = 32'h0000_0000
)(
    input  wire        clk,
    input  wire        rst,
    input  wire [31:0] next_pc,
    output reg  [31:0] pc
);

always @(posedge clk) begin
    if (rst)
        pc <= RESET_VECTOR;
    else
        pc <= next_pc;
end

endmodule
```
该模块使用：
```
高电平有效同步复位
上升沿状态更新
Non-Blocking Assignment
可配置Reset Vector
```
核心关系：
`Current State→Clock Edge→New State`