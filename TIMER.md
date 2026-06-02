adb push rtthread.bin /system/fw/rtthread_m7.bin

atool download -m rt_cm7 -f /system/fw/rtthread_m7.bin --ddr


共6个
gtest --gtest_filter=TIMER.RST.001.001    

gtest --gtest_filter=TIMER.TIMER.001.001   

gtest --gtest_filter=TIMER.TIMER.002.001

gtest --gtest_filter=TIMER.TIMER.004.001

gtest --gtest_filter=TIMER.TIMER.004.002

gtest --gtest_filter=TIMER.PAUSE.003.001  





devmem 0x03060008 32 0x10
devmem 0x03060000 32 0xF4240
devmem 0x03060008 32 0x1

devmem 0x03061008 32 0x10
devmem 0x03061000 32 0xF4240
devmem 0x03061008 32 0x1


devmem 0x42024014 32 0x01234567
devmem 0x42024884 32 0x1

devmem 0x43060008 32 0x10
devmem 0x43060000 32 0xF4240
devmem 0x43060008 32 0x1

devmem 0x43061008 32 0x10
devmem 0x43061000 32 0xF4240
devmem 0x43061008 32 0x1
devmem 0x43061010 32


// #define PERI_TIMER_MISC_OFFSET 0x14
// #define PERI_TIMER_MISC_OFFSET 0x18
#define PERI_TIMER_MISC_OFFSET 0x1c
// #define PERI_TIMER_MISC_OFFSET 0x20
