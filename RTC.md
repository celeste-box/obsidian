# 1. 概念
SOC支持一个RTC IP，位于aon_subsys中
RTC：real-time clock，独立工作的计数器
• 功能
1）用于休眠状态下的唤醒
2）提供周期性中断，用于实现软件对实时时钟的需求
3）维护绝对时间

• 包含两个计数器：
1）16-bit的pre-scaler计数器，预分频计数器，产生亚秒级时钟
输入：RTC 模块的源时钟（通常是 32.768 kHz 晶振，也可能是 RC 振荡器或外部时钟）。
输出：通过设置分频系数将高频源时钟降低到 1 Hz，馈送给时间计数器。
32.768 kHz 时钟 ÷ 32768 = 1 Hz，对应1s
2）32-bit的计数器，时间计数器，
输入：来自预分频器输出的 1 Hz 秒脉冲。
输出/作用：每个秒脉冲使计数器加 1，因此它的值表示从某个起始时刻（通常是 Unix 纪元或系统上电/复位后）经过的秒数。
支持写，如果发现当前时间无效后，完成时间校准获取当前正确时间写入RTC

RTC 周期性中断：用于“长周期、低功耗、能唤醒系统”的场景。
软件周期性定时器：用于“短周期、高


# 2. 验证
打开  IM_USING_RTC  和 TEST_RTC  开关后，M85核起不来
![[file-20260722164429837.png]]

报错原因：
![[file-20260722171135973.png]]
216行断言意思是：初始化 soft RTC 前，系统里不应该已经存在名为“rtc”的设备，根据warning 现在大概率是硬件rtc和soft rtc同时启动了，系统当前只允许一个RTC设备叫“rtc”
IM 硬件 RTC 驱动先注册 "rtc"
-> RT-Thread soft RTC 初始化
-> dev_soft_rtc.c 里检查 rt_device_find("rtc")
-> 发现已经有 "rtc"
-> RT_ASSERT(!rt_device_find("rtc")) 失败
解决方案：关掉CONFIG_RT_USING_SOFT_RTC
## 2.1 gtest --gtest_filter=RTC.RST.001  
验证复位功能
![[file-20260722164249776.png]]

## 2.2 gtest --gtest_filter=RTC.FUNC.003
验证分频功能
![[file-20260722164315009.png]]
## 2.3 gtest --gtest_filter=RTC.INTR.001
验证中断功能
![[file-20260722164349897.png]]