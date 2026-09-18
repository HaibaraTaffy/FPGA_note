第一版 Register File为：
```
32 × 32-bit
2R1W
组合读取
时钟沿写入
```
主要接口：
```
read_addr1 → read_data1
read_addr2 → read_data2

write_addr
write_data
write_enable
clk
```
写入条件：
```
always @(posedge clk) begin
    if (write_enable && (write_addr != 5'd0))
        registers[write_addr] <= write_data;
end
```
读取 `x0`永远返回0，写入 `x0`的操作被忽略：
```
assign read_data1 =
    (read_addr1 == 5'd0) ? 32'b0 : registers[read_addr1];
```
核心结论：
**Register File保存Architectural State**