>参考资料:[紫光同创开发板使用教程(一)：debug用法_紫光fpga debug-CSDN博客](https://blog.csdn.net/taowei1314520/article/details/130533510)

## 步骤
---
1. 直接进行编译综合
2. 添加debug和测试管脚 Tools的Inserter 即在线逻辑分析仪
   ![[Pasted image 20260828155225.png]]
3. **一般就Next**
   其他介绍:
   **Boundary Scan Chain（边界扫描链）**：下拉框当前选为 `USER1`。边界扫描（JTAG）除了标准指令外，还提供用户自定义指令（USER1~USER4），用于内部逻辑与 JTAG 接口的通信。这里选择 `USER1` 表示调试核将使用 USER1 扫描链进行数据交互。
   **Select Jtag（选择 JTAG）**：当前选为 `Default JIAG`（默认 JTAG 接口）。
   **左下 Core Occupancy (Estimated)（资源占用预估表）** : 显示插入调试核后预计占用的 FPGA 硬件资源
   ![[Pasted image 20260828155303.png]]
4. 设置通道数和抓取深度等参数(Trigger Parameters)
   **一般只配置红框的地方 甚至只需要配置最后一个**
- 触发输入与触发单元设置
	**Number of Input Trigger Ports（输入触发端口数）**：红框标注，当前设为 `1`。表示你打算使用 1 个触发端口来接入需要监控的触发信号（最多支持 16 个）。
	**Number of Trigger Units（触发单元数）**：红框标注，当前设为 `1`。表示挂接在当前触发端口下的匹配单元数量为 1 个（所有端口的触发单元总数不能超过 16）。
	**Used As Data（用作数据）**：已勾选。表示当前触发端口的输入信号，同时也作为数据捕获端口的输入信号（即同一组信号既用来做触发条件，也用来被抓取显示波形）。
	**Match Type（匹配类型）**：当前为 `Basic w/edges`，支持基本的电平匹配和边沿触发。
- 捕获与存储设置
	**RAM Type（存储类型）**：当前为 `Block RAM (Default)`，表示使用 FPGA 内部的块 RAM 来缓存抓取到的波形数据。
	**Sample Depth（采样深度）**：红框标注，当前设为 `512`。这表示逻辑分析仪能存储 **512 个时钟周期**的数据。深度越大，能看到的历史波形越长，但消耗的片内 Block RAM 资源也越多。
	**Sample On（采样边沿）**：当前为 `Rising`（上升沿），表示在采样时钟的上升沿进行数据采样。
	**Data Same As Trigger**：已勾选，与上面的 Used As Data 呼应，简化连接配置。
- 触发条件设置
	**Max Number of Sequencer Levels（顺序触发最大等级）**：当前为 `1`。用于设置多级顺序触发（比如先满足条件 A，再满足条件 B 才触发抓波形），1 表示不使用多级顺序触发。

![[Pasted image 20260828160121.png]]
5. 设置信号连接(Net Connections)
- CLOCK PORT（时钟端口）
	- **clk0**（红框1标注）：调试核的采样时钟输入端。逻辑分析仪必须有一个基准时钟才能工作
- TRIGGER PORTS（触发端口)
	- **TriggerPort0**：触发输入端口。因为你在上一步设置了“输入触发端口数为 1”且“用作数据”，所以这里显示 TriggerPort0。你需要把想要监控并作为触发条件的信号（例如 LED 输出信号、计数器某一位等）连接到这里。
通过 **Modify Connections** 来进行连接 连接完点击左上角的保存按钮
![[Pasted image 20260828160759.png]]
6. 添加信号
   >注意!!!**防优化警告**：如果在弹出的信号列表里找不到你想抓的信号，说明该信号在综合时被软件优化掉了。需要在 Verilog 代码中给该信号加上防优化属性`/* synthesis PAP_MARK_DEBUG="true" */`，然后重新综合工程。

挑选需要的时钟和信号 拖动到TriggerPort和Clock处 搞定后保存可以关闭Inserter窗口
![[Pasted image 20260828162332.png]]

7. 返回主界面 重新编译工程 生成比特流
8. 使用Debugger 下载工程
   ![[Pasted image 20260828163036.png]]
9. 链接![[Pasted image 20260828163049.png]]
![[Pasted image 20260828163058.png]]
10. 可以调试JTAG 选择参数
- 顶部工具栏
	- - **JTAG Scan Rate（JTAG 扫描速率）**：当前设为 `1s`。表示软件每隔 1 秒通过 JTAG 自动扫描一次硬件并刷新传感器数据。
	- **Window Depth（窗口深度）**：当前设为 `16`。表示在右侧历史记录（HISTORY）中保留最近 16 次的采样数据。
- 左侧 Device（设备栏）
	-  **JTAG Chain**：显示了当前通过 JTAG 连接到的硬件设备。
	- **DEV:0 MyDevice0 (Logos2-PG2L...)**：代表实际连接的紫光同创 FPGA 芯片型号。
	- **ADC Console**：当前选中的功能控制台。
- 右侧主表格（ADC Console - DEV 0）这是核心数据显示区，监控了 FPGA 裸片上的物理传感器：
	- **SENSOR（传感器类型）**：
		- **ChipTemperature**：芯片内部结温传感器。
		- **VCC**：芯片核心电源电压传感器（VCCINT）。
		- **VCCA**：芯片辅助电源电压传感器（VCCAUX）。
- 左下角 (ADC Console)
	- **On-Chip Sensors（片上传感器）**：列出了芯片内部可用的传感器通道，当前显示 `CH:0 ChipTemperature`。
![[Pasted image 20260828163245.png]]
11. 点击左上角第二个 下载 然后选择bits 下载
![[Pasted image 20260828164042.png]]

12. 弹出新界面 点击Waveform就可以观察波形了
![[Pasted image 20260828164142.png]]
13.点击运行
![[Pasted image 20260828164201.png]]
![[Pasted image 20260828164222.png]]

>还是算简单的 操作步骤也不难 