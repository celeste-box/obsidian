# 1. 概念
工作原理：
1. 启动后开始倒计时：软件使能 WDT，它从一个初始值（如 30 秒）开始递减。
2. 正常工作时定期“喂狗”：软件在主循环或关键任务中，定期向 WDT 的“喂狗寄存器”写入特定值(0x76)，将计数器重置回初始值。
3. 异常时“狗叫”：如果软件因跑飞、死锁而无法及时喂狗，计数器会减到 0。此时 WDT 会触发系统复位（最常见）或中断（先警告，再不复位则强制复位）。

支持两种超时模式：
1）直接产生系统复位
2）产生一个中断，第二次超时发生时，产生系统复位

# 2. 验证
T2605支持5个WDT 用于ACPU/M85/ADSP
## 2.1 FPGA
## 2.2 TO






rtos

adb push rtthread.bin /system/fw/rtthread_m7.bin

atool download -m rt_cm7 -f /system/fw/rtthread_m7.bin --ddr

devmem 0x42024084 32 0xff0000

gtest --gtest_filter=WDT.RST.001.001  1
   
gtest --gtest_filter=WDT.FUNC.002.001    32s超时中断 + 16s超时中断+复位  1

gtest --gtest_filter=WDT.FUNC.003.001    32s超时中断 + 16s超时中断+复位 1

gtest --gtest_filter=WDT.FUNC.003.002    32s复位	1

gtest --gtest_filter=WDT.FUNC.004.001    62.5ms复位	1

gtest --gtest_filter=WDT.FUNC.004.002    64s复位 

gtest --gtest_filter=WDT.FUNC.004.003    1s复位

gtest --gtest_filter=WDT.FUNC.004.004    4s复位

gtest --gtest_filter=WDT.FUNC.004.005	 125ms复位	1

gtest --gtest_filter=WDT.FUNC.004.006	 128s复位

gtest --gtest_filter=WDT.FUNC.004.007	 2s复位

gtest --gtest_filter=WDT.FUNC.004.008	 8s复位

gtest --gtest_filter=WDT.FUNC.007.001	 32s复位

gtest --gtest_filter=WDT.FUNC.008.001    暂停5s，37s复位

gtest --gtest_filter=WDT.FUNC.008.002	 32s复位

gtest --gtest_filter=WDT.INTR.002.001   不具备测试条件



![[file-20260707161807773.png]]





