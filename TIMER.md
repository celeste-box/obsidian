# 1. 概念
功能：定时器，用于实现软件定时中断及唤醒功能，每个Timer包含8个独立的通道的32bit计数器
工作方式：从0开始往上数，数到它眼前的上限值，然后瞬间跌落回0，再重新往上数，循环
8个独立通道：1个计数器可用于8个独立的控制器端
Free-Running模式：计数器跑满，寄存器配置要求是1s
![[企业微信截图_17830608018740.png]]
用户自定义模式：软件设定计数器上限值

配置流程
![[file-20260608145610191.png]]

pclk：寄存器读写时钟
timer_clk：工作时钟，支持timer_clk关闭后 跑 free-running模式

# 2. 验证
T2605支持4个 LPAI TIMER，1个AON TIMER
要遍历一个timer的8个独立通道
要遍历不同的timer

1.  adb push m85_load_ddr.sh /tmp

2.  adb push rtthread_m85.bin /tmp

3.  dos2unix  [m85_load_ddr.sh](https://m85_load_ddr.sh)

4.  chmod +x [m85_load_ddr.sh](https://m85_load_ddr.sh)

5.  sh  [m85_load_ddr.sh](https://m85_load_ddr.sh)

## 2.1 gtest --gtest_filter=TIMER.RST.001.001    
**注意crg相关更新**
![[file-20260709162832711.png]]

## 2.2 gtest --gtest_filter=TIMER.TIMER.001.001   
![[file-20260709174521689.png]]

**问题**：找不到设备
![[file-20260709163827244.png|650]]

解决：a500timer驱动没有编译，以下3个开关都要打开
im/rtos/rtos/rt-thread/rt-thread-5.2.2/bsp/imv/libraries/drivers/hw_timer   驱动位置
![[file-20260709181929495.png]]

## 2.3 @ gtest --gtest_filter=TIMER.TIMER.002.001 
![[file-20260709174820263.png]]

## 2.4 @ gtest --gtest_filter=TIMER.TIMER.004.001    
![[file-20260709174848245.png]]

## 2.5 gtest --gtest_filter=TIMER.TIMER.004.002
![[file-20260709174923817.png]]
![[file-20260709174942040.png]]

## 2.6 @ gtest --gtest_filter=TIMER.PAUSE.003.001  
![[file-20260709175003566.png]]



## 2.2 TO



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
