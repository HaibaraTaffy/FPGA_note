## ALU职责
---
ALU负责根据 `alu_control` 对两个32-bit Operand Value进行运算。
第一版支持：
```
0000 → ADD
0001 → SUB
0010 → AND
0011 → OR
0100 → XOR
```
ALU属于组合逻辑：
```
Operand A
Operand B
ALUControl
     ↓
    ALU
     ↓
Result、Zero
```
它不保存状态，因此不需要 Clock和Reset。
## ALU核心RTL
---
```
always @(*) begin
    case (alu_control)
        ALU_ADD: result = operand_a + operand_b;
        ALU_SUB: result = operand_a - operand_b;
        ALU_AND: result = operand_a & operand_b;
        ALU_OR:  result = operand_a | operand_b;
        ALU_XOR: result = operand_a ^ operand_b;
        default: result = 32'b0;
    endcase
end

assign zero = (result == 32'b0);
```
`default`保证所有组合逻辑路径都对 `result`赋值，避免推导出 Latch。
核心结论：
**ALU只负责组合计算，不保存处理器状态**​