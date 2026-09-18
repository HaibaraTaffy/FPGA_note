
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

## 扩展