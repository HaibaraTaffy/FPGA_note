>参考网站:[VIVADO SDK的使用-CSDN博客](https://blog.csdn.net/xiaoxian666/article/details/132210869)

## 什么是SDK ##
- Vivado SDK 是 Xilinx 提供的嵌入式软件开发环境，与 Vivado 工具链无缝集成，用于开发基于 FPGA 的嵌入式系统。
- 简单来说,就是在FPGA端负责软件部分开发的环境,在Zynq中,用于对arm端进行配置

## 开发流程 ##
1. ==**先设计好PL端代码**== 
   代码中需要包含**ZYNQ7 Processing System** 这个IP核 该IP核可能的创建流程如下:
	1. 在左侧导航栏（ Flow Navigator ）中，单击 IP Integrator 下的 Create Block Design 。然后在弹出的对话框中指定所创建的 Block Design 的名称，在 Design name 栏中输入“ system ”。
	2. 点击“ OK ”按钮，注意右侧的 Diagram 窗口，我们将在该窗口中以图形化的方式完成设计。
	3. 接下来在 Diagram 窗口中给设计添加 IP 。点击上图中箭头所指示的加号“ + ”，会打开 IP 目录（ IP Catalog）。也可以通过快捷键 Ctrl + I ，或者右键点击 Diagram 工作区中的空白位置，然后选择“ ADD IP ”。 ![[Pasted image 20260729145418.png]]
	4. 打开 IP 目录后，在搜索栏中键入“ zynq ”，找到并双击“ ZYNQ7 Processing System ”，将 ZYNQ7 处理系统 IP 添加到设计中。
	5. 添加完成后，ZYNQ7 Processing System 模块出现在 Diagram 中
	6. 双击所添加的 ZYNQ7 Processing System 模块，进入 ZYNQ7 处理系统的配置界面。界面左侧为页 面导航面板，右侧为配置信息面板。 ![[Pasted image 20260729145436.png]]
	7. ==重点==
		 - 在 Zynq Block Design 页面，显示了 Zynq 处理系统（ PS ）的各种可配置块，其中==灰色部分==是固定的， ==绿色部分==是可配置的，按工程实际需求配置。可以直接单击各种可配置块（以绿色突出显示）进入相应的 配置页面进行配置，也可以选择左侧的页导航面板进行系统配置。
		 - **PS-PL Configuration** 页面 用于配置PS-PL接口,包括AXI HP和ACP总线接口
		 - **Peripheral IO Pins** 页面 用于为不同外设选择MIO/EMIO配置
		 - **MIO Configuration** 页面 用于配置PS输入时钟 外设时钟 DDR和CPU时钟等
		 - **DDR Configuration** 页面 用于设置PS端DDR控制器配置信息
		 - ==?==**SMC Timing Calculation** 页面用于执行SMC时序计算
		 - **Interrupts** 页面用于配置PS-PL中断端口
	8. 具体配置此处不展开 要用的时候可以查阅相关手册 配置完毕点击右下角OK
    9. ![[Pasted image 20260729150543.png]]
    10. 点击上图中箭头所指**Run Block Automation**,弹出下图所示对话框
    11. ![[Pasted image 20260729150625.png]]
    12.  在该界面中我们可以选择自动连接 IP 模块的接口，包括导出外部端口，甚至可以自动添加模块互联过程中所需的IP.这样 **ZYNQ7 Processing System** IP核创建完毕
2. ==**导出至SDK**==
	1. ![[Pasted image 20260729150920.png]]
	2. 保存后,验证设计![[Pasted image 20260729151039.png]]
	3. 在 Sources 窗口中，选中 Design Sources 下的 sysetm.bd, 这就是我们刚刚完成的 Block Design 设计。 右键点击 sysetm.bd ，在弹出的菜单栏中选择 **“ Generate Output Products ”**，如上图所示：
	4.  在对话框中 Synthesis Options 选择 Global ； Run Setings 用于设置生成过程中要使用的处理器的线程数，进行多线程处理，保持默认或设置为个人电脑处理器最大可使用线程数都可以，一般选择最大可使用线程数的一般。然后点击“Generate ”来生成设计的综合、实现和仿真文件。
	5. 在“Generate ”过程中会为设计生成所有需要的输出结果。比如 Vivado 工具会自动生成处理系统的 XDC 约束文件，==因此我们不需要手动对 ZYNQ PS 引出的接口（DDR 和 FIXED_IO）进行管脚分配。==Generate 完成后，在弹出的对话框中点击“OK ”。
	6. 在“ Hierarchy ”标签页再次右键点击 system.bd ，然后选择 **“ Create HDL Wrapper ”** 。在弹出的对话框中确认勾选“Let Vivado manage wrapper and auto-update”，然后点击“ OK ”。![[Pasted image 20260729154635.png]]
	7. 在菜单栏选择 File > Export > Export hardware。![[Pasted image 20260729154955.png]]
	8. 在弹出的对话框中，没有生成 bitstream 文件，就无需勾选“ Include bitstream ”，若有,则需要勾选 之后点击OK![[Pasted image 20260729155213.png]]
	9. 在上图中，因为选择了“Export to Local to Project” ， Vivado 工具会在当前工程目录下新建一个文件夹， 名为“xxx(工程名).sdk” ，它是我们接下来软件开发的工作空间。 在 Export Hardware 的过程中，工具会将硬件以一个 ZIP 压缩文件的形式导出到该工作空间中，文件名为“system_wrapper.hdf” 。该文件包含了我们前面所搭建的硬件平台的配置信息，其后缀名 .hdf 的含义为“Hardware Definition File”，即硬件定义文件。
	10. 导出后 可以在File中选择Launch SDK 启动SDK开发环境
3. ==**从 Vivado 启动 SDK（2019.3）**==
	1. 在 Vivado 中完成 **Generate Bitstream** 后，选择 **File → Export → Export Hardware**，并勾选 **Include bitstream**，点击 **OK**。此步骤会更新工程的 `.hdf` 硬件描述文件；硬件有任何改动时都应重新导出。
	2. 保持 Vivado 工程打开，选择 **File → Launch SDK**。首次启动会弹出 *Launch SDK* 对话框：
		- **Exported location**：通常保持 Vivado 自动填写的当前工程 `.sdk` 目录；
		- **Workspace**：建议保持默认的 `工程名.sdk`，或指定一个专用且空的 SDK 工作区；不要选到旧版本硬件的工作区。
	3. 点击 **OK**，Vivado 会启动与 **2019.3** 匹配的 Xilinx SDK，并将刚导出的 `.hdf` 作为硬件平台带入该工作区。后续创建 Application Project 时，选择这个硬件平台。
	4. 若 SDK 已经打开，先在 Vivado 重新 **Export Hardware**，再从 Vivado 执行 **File → Launch SDK**，或在 SDK 中更新硬件平台和 BSP；不要只替换 `.bit` 文件。
4. SDK烧录

> [!important] 先区分两件事
> 本节的 **Xilinx Tools → Program FPGA** 是通过 **JTAG 临时下载**：配置 PL，并把 SDK 编译出的 ELF 下载到 PS 的 DDR 后运行。断电、复位或重新上电后内容会丢失；若要上电自启动到 QSPI/SD 卡，请见文末“固化启动”。

1. **烧录前检查**
	1. **生成 bitstream 并重新导出硬件**：回到 Vivado，依次执行 *Generate Bitstream* → *File → Export → Export Hardware*，并勾选 **Include bitstream**；然后重新启动 SDK，或在 SDK 中更新 Hardware Platform。没有 `.bit` 文件时，SDK 无法配置 PL。
	2. **确认 SDK 工程可正常编译**：在 *Project Explorer* 中右击应用工程，选择 **Build Project**。编译成功后会生成 `应用工程名.elf`（通常位于 `Debug/` 目录）。
	3. **连接并上电开发板**：连接 JTAG（通常为 USB/JTAG 口）和串口；开发板上电，并将启动模式拨到 **JTAG**。打开设备管理器确认 JTAG 驱动及串口均已识别。
	4. 如设计使用串口输出，打开串口终端，参数须与 Zynq PS 配置一致（常见为 `115200 8N1`）。

2. **使用 SDK 通过 JTAG 下载并运行**
	1. 启动 SDK 后，在 *Project Explorer* 中选中要运行的**应用工程**；确认它关联的是当前硬件平台/BSP，避免地址映射或驱动与 `.hdf` 不一致。
	2. 选择 **Xilinx Tools → Program FPGA**。在弹窗中检查：
	   - **Hardware Platform**：当前工程导出的硬件平台；
	   - **Bitstream**：当前硬件对应的 `.bit` 文件；
	   - **ELF File to initialize in block RAM**：仅当 MicroBlaze 程序需预初始化 BRAM 时选择 `.elf`；Zynq PS 的普通应用通常留空。
	3. 点击 **Program**。SDK 通过 JTAG 下载 bitstream 配置 PL；在底部 *SDK Log* 确认没有 `ERROR`。
	4. 右击应用工程，选择 **Run As → Launch on Hardware (System Debugger)**，或点击绿色运行按钮。首次按向导选择目标处理器（Zynq 通常为 `ps7_cortexa9_0`），其余保持默认，点击 **Run**。
	5. SDK 会将 `.elf` 下载到 DDR 并开始执行。在串口终端查看 `Hello World`、调试信息或应用输出；也可在 *Debug* 视图中断点调试。

> [!tip] 日常只改 C/C++ 软件时
> PL 硬件设计和 bitstream 未变化时，通常无需再次 **Program FPGA**；重新 **Build Project** 后执行 **Run As → Launch on Hardware** 即可。若板卡刚上电、复位过，或 PL 设计已更新，则需重新 Program FPGA。

### 4.3 常见问题排查

- **找不到 JTAG target / Program 失败**：检查供电、启动模式是否为 JTAG、USB/JTAG 线及驱动；关闭可能占用 JTAG 的 Vivado Hardware Manager 后重试。
- **Program 成功但串口无输出**：确认运行的是应用工程而非 BSP；检查串口号和 `115200 8N1`；按复位键后重新执行 *Launch on Hardware*。
- **程序跑飞或外设访问异常**：往往是 SDK 使用旧 `.hdf`/BSP。重新导出硬件（勾选 Include bitstream），更新硬件平台、重新生成 BSP，再编译应用。
- **只下载了 ELF，PL 外设不可用**：先执行 **Program FPGA**，确保当前 `.bit` 已下载；纯 PS 程序若不依赖 PL，则可不配置 PL。

### 4.4 固化启动（可选）

JTAG 下载不等于烧写 Flash。若要断电后自动运行，需要生成启动镜像并写入启动介质：

1. 在 SDK 中创建/编译 **FSBL**（*File → New → Application Project*，模板选择 *Zynq FSBL*），并准备应用 `.elf`；如需 PL，准备匹配的 `.bit`。
2. 使用 **Xilinx Tools → Create Boot Image**，按顺序加入：`FSBL.elf`（必须标记为 `bootloader`）→ `system_wrapper.bit`（可选）→ `app.elf`，生成 `BOOT.BIN`。
3. 将 `BOOT.BIN` 写入 QSPI Flash 或复制到 FAT32 格式 SD 卡根目录，并把启动模式切换为对应的 **QSPI** 或 **SD**；重新上电验证。

> [!warning] 固化前请确认开发板型号、Flash 类型/容量和启动模式。写错 QSPI 配置可能导致板卡暂时无法正常启动；保留可用的 JTAG 恢复方式。

参考：[Xilinx UG1138《Generating Basic Software Platforms》](https://www.amd.com/content/dam/xilinx/support/documents/sw_manuals/xilinx2019_1/ug1138-generating-basic-software-platforms.pdf)；[SDK 的 Program FPGA 操作说明](https://wiki.analog.com/resources/fpga/xilinx/software_setup)。