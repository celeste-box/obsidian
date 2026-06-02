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









