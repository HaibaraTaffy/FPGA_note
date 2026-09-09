>参考资料 : [【数字IC基础】跨时钟域（CDC，Clock Domain Crossing）-CSDN博客](https://blog.csdn.net/claylovetoo/article/details/129140192)

## 同步 异步设计
---
- 同步设计 : 在同步设计中  整个设计用的都是同一个时钟源
- 异步设计 : 在设计中 有**两个或两个以上的时钟** 且时钟之间是 `同频不同相`或`不同频`的关系 
  异步时序设计的关键就是把数据或控制信号正确地进行跨时钟域传输

## 跨时钟域CDC
---
跨时钟域问题的本质 : **亚稳态** 
即数据无法在规定的时间段内达到一个稳定的状态

可以根据传输的数据大小 分为
- 单比特亚稳态
- 多比特亚稳态
- 组合逻辑竞争冒险
- 时序逻辑亚稳态

亚稳态的发生原因:
1. 数据传输中 不满足`建立时间`和`保持时间`要求
2. 复位过程中 复位信号的释放 相对于有效时钟沿的`恢复时间`和`溢出时间`不满足
亚稳态的输出不稳定 但是会向后传播 导致出错 所以亚稳态危害很大

## 单bit信号的CDC
---
对于**电平信号**
`用两级D触发器做同步处理`
情景 时钟域A中的组合逻辑信号传到时钟域B中
需要注意的点 A中的组合逻辑信号 先在本时钟域(A)中打一拍(经过一级D_FF) 等数据稳定后 再传输到时钟域B 再打一拍 才能使用
![[Pasted image 20260909101147.png]]
```Verilog
//单比特电平信号 慢到快 示例
//因为电平信号 持续时间还算长 更何况是慢到快 所以可以通过这种持续检测来进行同步
//在同步完全前 Signal_in一般是不拉低的 
module single_cdc(
	input clk1,
	input clk2,
	input rst_n,
	input signal_in,
	output signal_out
);
reg signal_out_r;  //打一拍
reg signal_out_rr;  //打两拍

always @(posedge clk2 or negedge rst_n) begin
	if(!rst_n) begin
		signal_out_r  <= 1'b0;
		signal_out_rr <= 1'b0;
	end
	else if(signal_in == 1'b1) begin
		signal_out_r <= signal_in;
		signal_out_rr <= signal_out_r;
	end
	else begin
		signal_out_r <= 1'b0;
		signal_out_rr <= 1'b0;
	end
end

assign signal_out = signal_out_rr;

endmodule
```

对于**脉冲信号** 即数据只稳定一个时钟周期
可以分类讨论 这里先给出概述 :
1. 慢到快 : 先用**两级D触发器同步** 再用**边沿检测电路**得到脉冲信号
2. 快到慢 : 先将**脉冲信号展宽** 再**同步**到慢时钟域 最后用**边沿检测电路**得到脉冲信号


## 单bit脉冲 慢到快
---
即从慢时钟域同步到快时钟域 其实对于快时钟域 慢时钟域的信号可以近似视作电平

```Verilog
//单比特脉冲信号:慢到快
module single_cdc(
	input clk1,
	input clk2,
	input rst_n,
	input signal_in,
	output reg signal_out
);
reg signal_out_r;  //打一拍
reg signal_out_rr;  //打两拍
reg signal_out_rrr;  //边沿检测电路
always @(posedge clk2 or negedge rst_n) begin
	if(!rst_n) begin
		signal_out_r  <= 1'b0;
		signal_out_rr <= 1'b0;
	end
	else if(signal_in == 1'b1) begin
		signal_out_r <= signal_in;
		signal_out_rr <= signal_out_r;
		signal_out_rrr <= signal_out_rr;
	end
	else begin
		signal_out_r <= 1'b0;
		signal_out_rr <= 1'b0;
		signal_out_rrr <= 1'b0;
	end
end
//上升沿检测：检测 signal_out_rr 的上升沿，输出单时钟周期脉冲
assign signal_out = signal_out_rr && !signal_out_rrr;

endmodule

```

## 单bit脉冲 快到慢
---
采用脉冲展宽的方法 把快时钟域的信号多稳定一段时间 等到慢时钟域采到了 再拉低
由此可以保证数据的跨时钟域传输

1. 快时钟域 当采样到输入信号为高 拉高脉冲展宽信号
2. 脉冲展宽信号 在慢时钟域过两级DFF 确保慢时钟域采样到脉冲展宽信号 随后将信息返回到快时钟域(慢到快 信号在快时钟域过两级DFF 反馈一下)
3. 跨时钟域采样到慢时钟域返回的高电平 (表示慢时钟域已经成功采样到输入信号) **将脉冲展宽信号拉低** 慢时钟域采样到脉冲展宽信号为低 则拉低慢时钟域信号
4. 慢时钟域用边缘检测电路得到脉冲信号
![[Pasted image 20260909110445.png]]

看图时 注意上升沿 然后下一拍
代码示例
```Verilog
module led(
	input clk_fast,
	input clk_slow,
	input rst_n,
	input signal_in,
	output signal_out
);
//快时钟域脉冲展宽
reg signal_a;
always @(posedge clk_fast or negedge rst_n) begin
	if(!rst_n)
		signal_a <= 1'b0;
	else if(signal_in == 1'b1)//拉高
		signal_a <= signal_in;
	else if(signal_a_rr == 1'b1)//拉低
		signal_a <= 1'b0;
	//其它情况，保持上一时刻的值，这里可以省略
end
//慢时钟域采集脉冲展宽信号
//信号同步不用加判断条件
reg signal_b;  //打一拍
reg signal_b_r;  //打两拍
always @(posedge clk_slow or negedge rst_n) begin
	if(!rst_n) begin
		signal_b  <= 1'b0;
		signal_b_r <= 1'b0;
	end
	else  begin
		signal_b <= signal_a;
		signal_b_r <= signal_b;
	end
end
//快时钟域采集慢时钟域返回信息：signal_b_r
reg signal_a_r;  //打一拍
reg signal_a_rr;  //打两拍
always @(posedge clk_fast or negedge rst_n) begin
	if(!rst_n)
		{signal_a_rr,signal_a_r} <= {2{1'b0}};
	else
		{signal_a_rr,signal_a_r} <= {signal_a_r,signal_b_r};
		//左边给左边，右边给右边 移位的一种写法
end
//慢时钟域边沿检测，得到脉冲信号
//与逻辑检测上升沿：signal_b_r为1，且signal_b_rr为0
reg signal_b_rr; //上升沿检测,将signal_b_r打一拍
always @(posedge clk_slow or negedge rst_n) begin
	if(!rst_n)
		signal_b_rr  <= 1'b0;
	else 
		signal_b_rr <= signal_b_r;
end	

assign signal_out = signal_b_r && (!signal_b_rr);

endmodule

```



## 两级触发器的原因(扩展)
---
引入一个概念 MTBF (Mean Time Between Failure) 即平均失效间隔时间
**MTBF即触发器采样失败的时间间隔 MTBF时间越长 出现亚稳态的概率就越小 但是不能完全避免亚稳态**
![[Pasted image 20260909105326.png]]
经过一级DFF 相当于乘一个MTBF 即每过一个DFF 总MTBF会变大 亚稳态概率越小
![[Pasted image 20260909105333.png]]
在基本条件固定的情况下 想要增大MTBF 只能增大Tmet 回想起时序分析 往Pipeline中插入FF 就可以增大Tmet 从而增大MTBF

有文献给出 对于一个采样频率200Mhz的系统 如果
- 不做同步 MTBF = 2.5us
- 做一级DFF同步 MTBF = 23年
- 做两级DFF同步 MTBF = 640年
二级就够用了 三级可能影响系统性能 增加面积 没什么必要

## 多bit信号的CDC
---
即**数据收敛** 如何确保一组相关联的同步信号在经过**不同的路径之后可以在某一个相同的时钟周期正确地**到达另一个时钟域

如果还使用刚刚的双FF进行信号同步 就会使信号的准确性和关联性出现问题

可以采用 `格雷码编码 握手协议 FIFO`来解决

## 多bit信号 格雷码+双DFF
---
常用于异步FIFO中读写地址的跨时钟域传递
由于格雷码相邻状态只有1bit不同 所以同一时刻改变的只有一个 可以很好的避免多个位翻转 即多bit亚稳态的原因
了解一下即可 基本上在IP内使用 不是用户所关心

## 多bit信号 握手协议
---
握手协议 **将多bit数据的传输问题 转换成 单个信号的跨时钟域问题** 只对 `请求信号REQ` 和 `应答信号ACK` 进行同步

在REQ有效期间 发送端的数据保持不变 因此握手协议可以满足并行数据传输安全的需要 具体流程 *假设发送端clk_a 接收端 clk_b*
1. 类似AXI的vaild ready信号 发送端先将多比特数据驱动到总线上(准备好数据) 然后发送REQ
2. 接收端识别REQ 有效 接收这组数据 (**这里只需要将REQ同步到接收端时钟域**)  *两个clk_b同步*
3. 接收完毕后 接收端返回一个应答信号ACK *一个clk_b拉高*
4. 发送端识别到ACK有效 则将REQ拉低(**这里需要将ACK同步到发送端时钟域**) *两个clk_a同步 一个clk_a拉低*
   ![[Pasted image 20260909112158.png]]

代码示例
```Verilog
module led(
	input clk_a,
	input clk_b,
	input rst_n,
	input a_en,
	input [3:0] data_in,
	output b_en, 
	output reg [3:0] data_out
);
//a_en下降沿检测（与逻辑）
reg a_en_d1;
wire a_en_neg;
always @(posedge clk_a or negedge rst_n) begin
	if(!rst_n)
		{a_en_neg,a_en_d1} <= {2{1'b0}};
	else 
		a_en_d1 <= a_en;
end
assign a_en_neg = a_en_d1 && !a_en;
//a时钟域发出请求信号
reg req_a;
//ack信号打两拍同步到a时钟域
reg ack_a_r;
reg ack_a_rr;
always @(posedge clk_a or negedge rst_n) begin
	if(!rst_n)
		req_a <= 1'b0;
	else if(a_en_neg)
		req_a <= 1'b1;
	else if(ack_a_rr)//拉低
		req_a <= 1'b0;
	//其它情况，保持上一时刻的值，这里可以省略
end
//请求信号打两拍同步到b时钟域
reg req_b_r;
reg req_b_rr;
always @(posedge clk_b or negedge rst_n) begin
	if(!rst_n)
		{req_b_rr,req_b_r} <= {2{1'b0}};
	else
		{req_b_rr,req_b_r} <= {req_b_r,req_a};
end
//检测到a时钟域发出的请求信号的上升沿，则b时钟域可以接收数据
assign b_en = req_b_r && !req_b_rr;
//b时钟域接收数据
always @(posedge clk_b or negedge rst_n) begin
	if(!rst_n) 
		data_out <= 'b0;
	else if(b_en)
		data_out <=  data_in;
end
//b时钟域接收完数据，发送ack信号,打两拍同步到a时钟域
always @(posedge clk_a or negedge rst_n) begin
	if(!rst_n)
		{ack_a_rr,ack_a_r} <= {2{1'b0}};
	else
		{ack_a_rr,ack_a_r} <= {ack_a_r , req_b_rr};
end

endmodule

```

## DMUX(DFF加一个二选一MUX) 数据使能选通设计
---
通过一个使能信号 来判断data信号是否已经稳定
使能信号有效 说明data属于稳定状态 在这种情况下 终点寄存器才对信号进行采样
**将多比特信号的CDC转换成单比特信号的CDC**

慢到快 快到慢参考单比特

使能信号接MUX的sel端 

若使能信号有效 则选通发送端数据
若无效 则选通'接收端数据'
**注意 MUX的输出端无法直接与输入端相连 需要先将输出数据保存在DFF中 再与输入端相连**
![[Pasted image 20260909112723.png]]

```Verilog
//DMUX
module led(
	input clk_a,
	input clk_b,
	input rst_n,
	input a_en,
	input [3:0] data_in,
	
	output reg [3:0] data_out
);
//a时钟域使能信号同步到b时钟域,作为MUX的sel
reg a_en_r;
reg a_en_rr;
always @(posedge clk_b or negedge rst_n) begin
	if(!rst_n)
		{a_en_rr,a_en_r} <= {2{1'b0}};
	else 
		{a_en_rr,a_en_r} <= {a_en_r,a_en};
end
//二选一MUX
always @(posedge clk_b or negedge rst_n) begin
	if(!rst_n)
		data_out <= 'b0;
	else if(a_en_rr == 1'b1)//如果使能信号有效
		data_out <= data_in;
	else //如果使能信号无效
		data_out <= data_out;
		//这句话就综合出了一个接收端的DFF 接在选通端
		
end

endmodule

```


## 多路扇出
---
有些情况下 一个信号在跨越时钟域后分成了多个分支
**同一个信号源经过不同的路径跨越时钟域后 多路扇出的值不一定相同**

可以通过 `先同步 后扇出`的方式解决
即在时钟域B经过两级DFF同步 再扇出
![[Pasted image 20260909102555.png]]

## 数据丢失
---
输入端信号不能保持足够的时间 使接收端不能采样到数据 导致数据丢失
可以通过`延长输入信号`的方式 解决这个问题
![[Pasted image 20260909102647.png]]
值得注意的是 这里采用的是 缓存后做或运算 可以延长信号持续时间

## 异步复位
---
由亚稳态的产生可知 异步复位信号在不满足恢复时间和移除时间的要求时 会导致亚稳态

- 恢复时间 : 在有效的时钟沿到来之**前** 触发器的异步复位信号释放时 所要**提前释放的最小时间**
  理解 : 要开始工作了 但是你还在休息(复位信号还没走) 那肯定是不行的
- 移除时间 : 在有效的时钟沿到来之**后** 触发器的异步复位信号释放时 所要**保持不变的最小时间**
  理解 : 工作后要休息(复位) 不休息够(复位保持一定时间) 不能继续工作

异步复位信号引起的触发器亚稳态 并不是在复位的时候出现的 而是在复位信号释放时出现的 原因在于 **复位信号释放时不能够与触发器的时钟保持同步**

解决办法 `在异步复位信号释放时对其进行同步处理`
![[Pasted image 20260909103518.png]]

