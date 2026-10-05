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

---
## 以下为扩展笔记 2026/9/17

## 增加指令
---
- SLL Shift Left Logical 逻辑左移
- SRL Shift Right Logical 逻辑右移
- SRA Shift Right Arithmetic 算术右移
- SLT Set Less Than 小于则置位

## 基本形式
---
```
sll rd, rs1, rs2
srl rd, rs1, rs2
sra rd, rs1, rs2
slt rd, rs1, rs2
```
**均属于 R-type**

## Shift Amount
---
移位量 常缩写为 `shamt`
对于 RV32I 移位量来自
`RF[rs2][4:0]` 即**rs2的低五位**

对于一个 32bit 的数据 有意义的移位量为 `0~31`
表示 0~31 只需要5 bit

## SLL 
---
逻辑左移 
例如：
```
RF[x1] = 0x00000003
RF[x2] = 2
```
执行：
```
sll x3, x1, x2
```
计算：
```
0x00000003 << 2
= 0x0000000C
```
因此：
```
RF[x3] = 0x0000000C
```
一般情况下 左移N位 等效于 乘以 2^N
在Verilog中 可以写成
`operand_a << operand_b[4:0]`

## SRL
---
**左侧高位补0 无论正负数**
**也是取 操作数b的低5位**
例如：
```
RF[x1] = 0x80000000
RF[x2] = 1
```
执行：
```
srl x3, x1, x2
```
计算：
```
0x80000000 >> 1
= 0x40000000
```
因此：
```
RF[x3] = 0x40000000
```
即使 `0x80000000` 的最高位是1，SRL 仍然在左侧补0。
在Verilog中 可以写成
```operand_a >> operand_b[4:0]```

## SRA
---
**左侧高位补符号位**
**算术右移负数时的舍入方向与 RISC-V 的有符号除法不一定完全相同 因此不能把所有 SRA 都简单等同于有符号除法**

## Verilog的Signed
---
当前 ALU 接口 为 `input wire [31:0] operand_a;`
没有声明`signed` 于是都当 `unsigned` 无符号类型处理

在实现 `SRA`时 应该明确将 `operand A` 按有符号数解释
于是示例
```Verilog
$signed(operand_a) >>> operand_b[4:0]
```

><< 左移 >>右移
><<< 算术左移 >>> 算术右移

对于有符号比较 也应该
```Verilog
result =
    ($signed(operand_a) < $signed(operand_b))
    ? 32'd1
    : 32'd0;
```

## SLT
---
**小于则置位**

SLT 执行**有符号**比较：
```
如果 signed(RF[rs1]) < signed(RF[rs2])
    RF[rd] = 1
否则
    RF[rd] = 0
```
比较成立时，写入的是完整的32位数值：
```0x00000001```
比较不成立时，写入：
```0x00000000```

SLT 不是只产生一个1-bit结果 它最终仍然通过 Write-Back MUX 向 `RF[rd]` 写入32-bit数据

## Instruction Encoding
---
四条指令都属于 R-type 且 Opcode相同 均为`0110011`

| 指令    | `funct7`  | `funct3` | `opcode`  |
| ----- | --------- | -------- | --------- |
| `sll` | `0000000` | `001`    | `0110011` |
| `slt` | `0000000` | `010`    | `0110011` |
| `srl` | `0000000` | `101`    | `0110011` |
| `sra` | `0100000` | `101`    | `0110011` |
其中：
- `sll` 通过 `funct3=001` 识别
- `slt` 通过 `funct3=010` 识别
- `srl` 和 `sra` 的 `funct3` 都是 `101`
- `srl` 与 `sra` 需要进一步检查 `funct7[5]`

当前CPU已经提取 `funt7[5]`

## 扩展 ALU Control 编码
---
现有 ALU Control 编码
```
0000 → ADD
0001 → SUB
0010 → AND
0011 → OR
0100 → XOR
新增:
0101 → SLL
0110 → SLT
0111 → SRL
1000 → SRA
```

