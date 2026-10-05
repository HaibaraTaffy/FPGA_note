
## JAL
---
Jump And Link 跳转并保存返回地址
**无条件跳转**
```
jal rd offset
```
一次干两件事
```
PC ← current_pc + offset 跳转
RF[rd] ← current_pc + 4 保存返回地址
```

例如 :
```
当前 PC = 0x0000_0100
执行 jal x1 +16
之后

x1 = 0x0000_0104
PC = 0x0000_0110
```
以后程序执行完某个函数之后 可以根据这个返回地址回来
(就是把这个地址重新存入PC?)

## 扩展 Write-Back MUX
---
现有代码
```Verilog
assign write_back_data =
result_src ? memory_read_data : alu_result;
```

但是 JAL 也要往寄存器写入 `PC + 4`
于是需要扩展为
```
                   ┌─ ALU Result
                   │
Write-Back MUX ────┼─ Memory Read Data
                   │
                   └─ PC + 4
```

将 `result_src` 扩展成 2 bit
进行编码

|`result_src`|写回来源|
|---|---|
|`00`|ALU Result|
|`01`|Memory Read Data|
|`10`|PC + 4|
|`11`|暂时保留|
**数据通路设计思维
控制信号宽度不是固定的
它取决于需要多少条 Data Path**

## J-type 的立即数
---
在 J-type中
```
31          30:21       20        19:12
┌──────────┬───────────┬─────────┬───────────┐
│ imm[20]  │ imm[10:1] │ imm[11] │ imm[19:12]│
└──────────┴───────────┴─────────┴───────────┘
```

于是重新拼接 RTL设计如下
```Verilog
immediate = {
    {11{instruction[31]}},
    instruction[31],
    instruction[19:12],
    instruction[20],
    instruction[30:21],
    1'b0
};
```

**由于 RISC-V 的 J Immediate 本身就以 2 Byte 为单位编码 用于扩展16bit的指令 所以最低位天然为0**

设定 `imm_src = 11` 对应 IMM_J

## JAL的路径
---
```
Instruction
      │
      ▼
   ImmGen
      │
      ▼
J Immediate
      │
      ├────────────────┐
      │                │
current_pc        current_pc
      │                │
      ▼                ▼
   + immediate        + 4
      │                │
      ▼                ▼
 Jump Target         PC+4
      │                │
      ▼                ▼
 Next-PC          Write-Back MUX
      │                │
      ▼                ▼
     PC              RF[rd]
```

## 扩展 Main Decoder
---
现有
```
reg_write
alu_src
mem_write
result_src
branch
imm_src
alu_op
```
新增一个 `jump`

| 控制信号         |  JAL |
| ------------ | ---: |
| `reg_write`  |  `1` |
| `mem_write`  |  `0` |
| `result_src` | `10` |
| `branch`     |  `0` |
| `jump`       |  `1` |
| `imm_src`    | `11` |
| `alu_op`     | `00` |
| `alu_src`    |  `0` |
由于不经过alu 于是 `alu_op` 和 `alu_src` 并未有任何作用

## 扩展 Next-PC Logic
---
现在代码
```Verilog
assign pc_src = branch & branch_condition;

assign next_pc =
pc_src ? branch_target : pc_plus4;
```

现在存在两个跳转情况 `branch成立` `Jump`

于是需要更新 `pc_src`
```Verilog
assign pc_src = 
jump | (branch & branch_condition);
```

原来的
```Verilog
assign next_pc =
pc_src ? branch_target : pc_plus4;
```
还可以继续使用
**但这里的 branch_target 已经不是只为 branch服务了**

## 上升沿读取
---
`pc` 在上升沿更新了 那写入 `RF[rd]` 的 `PC + 4` 会不会变成`新的pc + 4`

上升沿到来的时候 用的都是 之前已经计算好的值 不会出现这种冲突


---
## JALR
---
Jump And Link Register 寄存器间接跳转并保存返回地址

形式 ```
```
jalr rd immediate(rs1)
```
完成两件事 :
```
RF[rd] <- PC + 4 将下一条指令存入 rd 中
PC <- (RF[rs1] + Immediate) & ~1
将 rs1 中存储的地址 和 Sign-Extended 12bit 立即数相加 并且把最低位强制清零
```
**JALR 使用 I-type Immediate**
所以无须修改 ImmGen

## 举例
---
假设：
```
PC = 0x0000_0100
RF[x5] = 0x0000_0201
Immediate = 4
```
执行：
```
jalr x1  4(x5)
```
首先计算：
```
RF[x5] + Immediate
= 0x201 + 4
= 0x205
```
然后 JALR 有一个特殊规定：
```
Target[0] = 0
```
所以：
```
0x205
↓ 清除 bit[0]
0x204
```
同时：
```
RF[x1] = PC + 4

       = 0x104
```
最终：
```
PC ← 0x204
x1 ← 0x104
```

## 扩展 Main Decoder
---
新增 `jalr` 控制信号 为 0 表示不是 jalr指令 为 1 表示是 jalr 指令

新增 `jalr` opcpde = 1100111

| Signal       | JALR |
| ------------ | ---: |
| `reg_write`  |  `1` |
| `alu_src`    |  `1` |
| `mem_write`  |  `0` |
| `result_src` | `10` |
| `branch`     |  `0` |
| `jump`       |  `0` |
| `jalr`       |  `1` |
| `imm_src`    | `00` |
| `alu_op`     | `00` |
## 扩展 Next-PC Logic
---
通过复用 ALU 的计算结果
`RF[rs1] + Immediate`
得到需要的 `target`

最低位清零 可以用 `截断+补零` 的方式
```Verilog
assign jalr_target = {
    alu_result[31:1],
    1'b0
};
```

