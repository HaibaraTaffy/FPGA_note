> 本笔记基于已阅读的 Pango/Xilinx 光纤视频工程学习资料整理。

## 视频数据怎么走

像素流可以直接边来边走，也可以先丢进 DDR。

- HDMI 输入一般更像流式通路。
- 摄像头由于时钟不一样，经常是：像素域 → 写 FIFO → DDR → 读 FIFO → 输出域。

读写 FIFO 不是多余的，一个管写侧和 MIG 时钟，另一个管 MIG 读数据和显示时钟。

## MIG

MIG 的命令 ready 和写数据 ready 是两件事。读命令送出去以后也不能立刻认为数据来了，要等读数据 valid。

地址到底是 byte、pixel 还是 MIG 的数据拍必须想清楚。少一个像素会逐行积累，最后画面可能横移。

## 切换

不要把请求的 source 直接接到像素 MUX。先有 request，再在 VS 或 DMA 完成的时候更新 active。request 和 active 不同是正常的。