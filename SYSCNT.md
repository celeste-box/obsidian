# 1.概念
1. 64bit计数器，用于系统计数
2. 从芯片boot中使能计数，直到芯片断电后复位停止
3. 位于AON_SUBSYS，以广播形式供其他模块使用
4. 频率比校准：ratio=整数部分(int) + 小数部分(frac) / 4096(2^12)
   理论值：参考时钟f_ref=32.768kHz，工作时钟syscnt_clk=3.84MHz      默认ratio=3840000 / 37268 =117.1875
   **实际syscnt_clk = ratio * f_ref**
   寄存器配置的就是ratio
   ![[file-20260611171032916.png]]
5.  master：支持输出脉冲信号pps(单次/周期)，同时锁存syscnt值
6. slave：支持在输入脉冲信号pps的上升沿锁存syscnt值，同时产生一个pps锁存中断上报内核[产生脉冲 -> 锁存 -> 报中断 -> CPU 计算 -> 写入补偿 -> 下一次锁存误差减小]，master和slave两种模式互斥
7. **所有的feature都是对 s_syscnt而言的**
8. NS_SYSCNT复位后存在短暂清零，随后自动重新同步S_SYSCNT



https://gerrit.imv.local/c/rtos/rt-thread/rt-thread/bsp/tetras/+/69628
https://gerrit.imv.local/c/testing/tests/+/69591
https://gerrit.imv.local/c/testing/tests/+/62502/12/test_cfg/configs/e220d_desc.json
https://gerrit.imv.local/c/rtos/rt-thread/rt-thread/bsp/tetras/+/64328/7/libraries/drivers/syscnt/tetras_syscnt.c


# 2. 文件位置
![[file-20260612172058438.png]]

| Bit  | 数值     | 含义                                 |
| ---- | ------ | ---------------------------------- |
| bit0 | `0x01` | 频率比例 `freq_ratio` 计算完成             |
| bit1 | `0x02` | 上一次频率比例计算仍处于 busy，又发起了新的计算         |
| bit2 | `0x04` | 捕获到 `pps_i`，达到 PPS 计数阈值，产生锁存中断     |
| bit3 | `0x08` | `C1` 补偿仍 busy 时又到达 PPS 上升沿         |
| bit4 | `0x10` | 上一周期的 `C3` 尚未补偿完，新 PPS 又产生了新的 `C3` |
| bit5 | `0x20` | `C3` 测量下溢，配置的目标 `pps_period` 偏小    |
| bit6 | `0x40` | `C3` 测量上溢，配置的目标 `pps_period` 偏大    |
| bit7 | `0x80` | 保留                                 |
|      |        |                                    |
# 3.验证
1. 1颗soc芯片仅支持1个syscnt IP
2. 在FPGA中160K替代的是RTC晶振(32.768kHz)成为参考时钟
3. 读或写都是先低32bit，再高32bit
4. C1：微调绝对值补偿
5. C2：微调预补偿（周期性测量补偿）
6. C3：微调周期测量与补偿（根据周期信号来测量补偿）
# 3 T2605修改点
rtos/rtos/rt-thread/rt-thread-5.2.2/bsp/imv/libraries/drivers/syscnt/Kconfig
rtos/rtos/rt-thread/rt-thread-5.2.2/bsp/imv/a500_soc/a500_fpga/.config
rtos/rtos/rt-thread/rt-thread-5.2.2/bsp/imv/a500_soc/a500_fpga/rtconfig.h

## 3.1 FPGA

1.  adb push m85_load_ddr.sh /tmp

2.  adb push rtthread_m85.bin /tmp

3.  dos2unix  [m85_load_ddr.sh](https://m85_load_ddr.sh)

4.  chmod +x [m85_load_ddr.sh](https://m85_load_ddr.sh)

5.  sh  [m85_load_ddr.sh](https://m85_load_ddr.sh)

M85 LOAD DDR 命令:

atool download -m rt_cm85 -f /system/fw/rtthread_m85.bin -d 0x45000000

M85 LOAD TCM 命令

atool download -m rt_cm85 -f /system/fw/rtthread_m85_tcm.bin -i 0x00400000

### 3.1.1 gtest --gtest_filter=SYSCNT.003.001  回片后测试
### 3.1.2 gtest --gtest_filter=SYSCNT.005.001   **TO 阶段验证**   
硬件校准频率比是自动的，不需要配置
### 3.1.3 gtest --gtest_filter=SYSCNT.005.002   **TO阶段验证**    
先配置软件配置频率比，在使能
### 3.1.4 gtest --gtest_filter=SYSCNT.006.001 
复位测试
![[file-20260710153302825.png]]
### 3.1.5 gtest --gtest_filter=SYSCNT.011.001 
计数器使能
![[file-20260710153238905.png]]

问题：使能失败
![[file-20260710145233233.png]]
解决：基地址需要适配a500
![[file-20260710151714505.png]]

### 3.1.6 gtest --gtest_filter=SYSCNT.012.001
支持计数器load功能，允许软件加载起始值
![[file-20260710153515258.png]]
### 3.1.7 gtest --gtest_filter=SYSCNT.017.001   TO阶段验证
频率比硬件周期支持16ms

### 3.1.8 gtest --gtest_filter=SYSCNT.025.001
计数暂停测试，允许通过haltdbg输入信号
![[file-20260710162109336.png]]

问题：
![[file-20260710154824534.png]]

解决：在AON_SC中这个寄存器也需要配置，代码中相关变量需要连接定义
![[file-20260710160204891.png]]
![[file-20260710162155092.png]]


![[file-20260710160304480.png]]
要根据寄存器的描述，修改对应的KEY
![[file-20260710162256783.png]]

### 3.1.9 gtest --gtest_filter=SYSCNT.046.001
触发频率校准比错误中断：频率比校准过程中（未完成），软件第2次频率比校准的错误中断指示
![[file-20260710163349320.png]]

问题：测试失败
![[file-20260710162752463.png]]
解决：syscnt基地址和a500没对应上
### 3.1.10 gtest --gtest_filter=SYSCNT.029.001
master模式，pps_o信号单次触发
![[file-20260721181231670.png]]

程序随机跑死:
mdelay(11) 和 rt_thread_delay(11) 的区别大概是：
rt_thread_delay(11)：线程睡眠，让出 CPU，等调度器之后再把你唤醒；实际回来时间可能更晚。
mdelay(11)：忙等 11ms，不主动让出 CPU，通常比线程 delay 更“贴近 11ms”。但会造成系统挂死
![[file-20260710172447242.png]]
![[file-20260715112253425.png]]

解决：
将代码中的 mdelay替换成rt_thread_delay
对于rt_thread_delay的单位由RT_TICK_PER_SECOND决定，目前在rtconfig.h中定义
![[file-20260715111605377.png|656]]

### 3.1.11 gtest --gtest_filter=SYSCNT.029.002
master模式：pps_o信号周期性触发，每次获取的差值应该等于设置的周期
![[file-20260717152022872.png]]

每隔10个周期读取一次
![[file-20260722122810286.png]]

[以下用例需要两台FPGA对接，一个做master，一个做slave]
    writel(0, S_SYSCNT_BASE + 0x80);    // 解除外设屏蔽
    writel(0x4, S_SYSCNT_BASE + 0x84); // 触发置
    rt_thread_mdelay(10);
Master固定运行gtest --gtest_filter=SYSCNT.030.002
### 3.1.12  gtest --gtest_filter=SYSCNT.027.001   slave模式
slave模式，锁存syscnt的同时，会产生一个pps中断并上报内核


devmem 0x12029050 32读死机了，说明gpio的时钟没开或处于复位状态，驱动没加载
![[file-20260717155002360.png]]


### 3.1.13  gtest --gtest_filter=SYSCNT.030.001
slave 粗调，补偿3840000
后运行gtest --gtest_filter=SYSCNT.027.001  
![[file-20260814142346153.png]]
粗调量 = (D1-D0) - 正常PPS周期
       = 4,607,999 - 768,000
       = 3,839,999    仅相差 1 tick，符合预期
       
![[file-20260814180627540.png]]
![[file-20260814180639272.png]]

### 3.1.14  gtest --gtest_filter=SYSCNT.030.002     master用
中断触发间隔768000，大约0.2s
Master 和 Slave 锁存的是各自本地 SYSCNT，绝对值不要求相等

### 3.1.15  gtest --gtest_filter=SYSCNT.034.001
C1补偿
后运行gtest --gtest_filter=SYSCNT.027.001  
![[file-20260814142512463.png]]
59904意味着comp_period =1.95ms，硬件每隔1.95ms执行一次补偿动作，每次补偿动作的调度周期
C1 = 38400：所有补偿周期累计需要完成38400 tick ，C1 会被分散到多个补偿周期中，Comp[0] + Comp[1] + ... + Comp[n] = 38400


D1-D0 = 789483 = 768000 + 21483
D2-D1 = 784917 = 768000 + 16917
累计 C1：21483 + 16917 = 38400


![[file-20260814180729857.png]]
![[file-20260814180748857.png]]
### 3.1.16 gtest --gtest_filter=SYSCNT.035.001
C2软件补偿，
后运行gtest --gtest_filter=SYSCNT.027.001    
用于配置软件计算得到的 C2 固定频偏补偿
C2在每个补偿周期持续生效   
![[file-20260814142530950.png]]


补偿值0xFFFA(-6)，换算  -6/256=0.0234375 tick/周期
寄存器0x100
comp_period=10752，一个 PPS周期为 768000,
补偿次数 ≈ 768000 / 7488
         ≈ 102.56次

每个PPS的理论补偿量
≈ -0.0234375 × 102.56
≈ -2.404 tick/PPS

D0  = 780800976
D22 = 797696924
D22-D0 = 16895948
22 × 768000 = 16896000

累计补偿量 = -52 tick
平均周期   = 16895948 / 22
           ≈ 767997.636

平均补偿量 ≈ -2.364 tick/PPS

22个周期的理论累计补偿约 -52.9 tick，实际为 -52 tick
![[file-20260814180845705.png]]
![[file-20260817190659423.png]]
![[file-20260817190725217.png]]
![[file-20260817190737524.png]]

### 3.1.17  gtest --gtest_filter=SYSCNT.036.001
C2硬件补偿
只使能“硬件测量 PPS 周期偏差”，并不执行自动补偿
![[file-20260814142610444.png]]

读取0x104值
D2-D1 = 767999，偏差 +1
D3-D2 = 767998，偏差 +2
D4-D3 = 767998，偏差 +2
![[file-20260814181005831.png]]
### 3.1.18  gtest --gtest_filter=SYSCNT.038.001
C3补偿，用于配置 Slave，通过外部 PPS 自动测量周期偏差 C3，并在下一 PPS 周期对 SYSCNT 进行硬件补偿。（可适当增大采样周期）

D1-D0 = 1401343034 - 1400575037 = 767997
D2-D1 = 1402111032 - 1401343034 = 767998
D3-D2 = 1402879031 - 1402111032 = 767999
D4-D3 = 1403647032 - 1402879031 = 768001
-3 → -2 → -1 → +1

D4  = 1403647032
D37 = 1428991032

D37-D4 = 25344000
33 × 768000 = 25344000  也就是稳定后的33个 PPS周期平均值恰好为768000 tick/PPS

![[file-20260817191047701.png]]
![[file-20260817191108544.png]]

![[file-20260817191121020.png]]
![[file-20260817191136545.png]]

### 3.1.19  gtest --gtest_filter=SYSCNT.038.002
C1+C2+C3，组合补偿用例：先执行 C1 一次性偏差补偿、配置 C2 固定频偏补偿，再开启硬件 PPS 测量与 C3 动态补偿。
(复位后跑，避免残留的 C1/C2/C3和中断状态干扰)

D1-D0 = 1418183845 - 1417415847
      = 767998

D2-D1 = 768000
D3-D2 = 768000
...
D42-D41 = 768000

(D42-D0) / 42
= (1449671845 - 1417415847) / 42
≈ 767999.952  
与目标周期 768000 几乎完全一致。除第一个完整周期少 2 tick 外，从 D2开始持续稳定在 768000，说明联合补偿已经快速收敛，并且没有继续漂移。
![[file-20260817185013491.png]]
![[file-20260817185026850.png]]

![[file-20260817185038790.png]]
![[file-20260817185053363.png]]

### 3.1.20  gtest --gtest_filter=SYSCNT.048.001
下溢
 "SYSCNT.048.001": {"lrs_id":["LRS.SOCIP.SYSCNT.048"], "suit":"syscnt", "entry": "038_syscnt_ft_comp_c3_hw", "paras": "3840"}
![[file-20260814181507542.png]]

上溢
 "SYSCNT.048.001": {"lrs_id":["LRS.SOCIP.SYSCNT.048"], "suit":"syscnt", "entry": "038_syscnt_ft_comp_c3_hw", "paras": "3840000"}
![[file-20260814181800808.png]]
### 3.1.21  syscnt_sv ns_read
ns_syscnt和s_syscnt之间的值差
![[file-20260727113639517.png]]

ns_syscnt 位于 M85 子系统，属于本地访问；s_syscnt 位于 AON 子系统，M85 读取它需要经过系统互联、总线桥、安全检查和跨时钟域，因此访问路径更长。
软件又是先读 ns_syscnt、后读 s_syscnt，所以跨子系统访问耗时被直接计入 delta。
关中断后结果稳定在 22/23 tick，说明主要是固定的跨子系统总线延迟，不是后台线程或两个计数器不同步。

![[file-20260727195816226.png]]


### 3.1.22  syscnt_sv ns_reset
ns_syscnt复位测试机制：NS_SYSCNT复位后存在短暂清零，随后自动重新同步S_SYSCNT
M85刚好能抓到短暂清零
![[file-20260724180929780.png]]

M3
![[file-20260728200623271.png]]