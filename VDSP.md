# 1. 框图
![[file-20260602112532363.png]]
# 2. 概念
## 2.1 DVFS
动态电压频率调整技术。简单来说，就是芯片的“经济运行模式”:当需要处理复杂任务时，就提升频率和电压来获得强劲性能;当任务简单或待机时，
就降低频率和电压，达到省电的目的。
T2605频率和电压对应关系
1500    1248   1200     940.8   705.6  500     300MHz
0.85V   0.80V  0.75V   0.70V   0.65V  0.60V  0.55V

功耗P = C * V² * f（C为负载电容，V为电压，f为频率）
## 2.2 算力
1.5 TOPS INT8 \192 GFLOPS FPI6 \@0.85V
INT8 算力 1.5 TOPS 相当于每秒 1.5 万亿次 8 位整数运算(TOPS:每秒万亿)                               AI推理，图像处理
FP16 算力 180GFLOPS 浮点性能，相当于每秒1800 亿次16位浮点运算(GFLOPS:每秒10亿)      仿真，3D图形
## 2.3 memory
采用哈弗架构，指令和数据独立编址，具体的编址规格由 LSp(linker SsupportPackage)决定，而用户可以通过名为 memap.xmm 的内存配,置文件来定义和修
改LSP
![[file-20260602154045084.png]]
### 2.3.1 memory划分
DSP memory就分3个部分，cache(L1D L1I),  TCM(IRAM DRAM),   SRAM(DDR)
IRAM DRAM属于TCM(紧耦合内存)，TCM 是直接物理连接在处理器内核旁边的专用高频 SRAM
cache是由硬件自动管理的“智能中转站”，而 TCM 是由软件(程序员)完全掌控的“专属VIP机房”。
cache和TCM在硬件总线架构上是并列的，IRAM中存放的是最核心最内层的代码,会被频繁调用的代码；而L1I是把大量的放在DDR中业务逻辑代码，把VDSP需要执
行的那几句加载到L1I中
### 2.3.2 cache一致性问题
1)地址分流：CPU通过访问的物理地址(Memory yap)自动判断走哪条总线:TCU走内部专用道(绕过LD)，DDR走外部系统总线(触发L1D)，
2)全自动缓存：对于DDR访问，L1D开启全自动批发。命中(Ht)时直接用L1D副本;缺失(Miss)时去外部DDR拉取一整行(64字节的cache line)留存记
录。
3)一致性漏洞：由于iDM搬运数据属于硬件行为、绕过了CPU，导致“外部DDR/内部ICM的数据已经被iDMA刷新”，而“L1D小本本上还是历史旧副本”，产生
数据时差，导致Cache不一致。
4)软件纠偏(核心指令):
cache Invalid(作废);配合“DMA接收”。把L1D的有效位清零(撕账本)，强迫CPU下次必须去外圈物理主存(DDR)拿iDMA刚运来的新数。
Cache F1ush /Clean(回写):配合“DMA发送”。把L1D里的CPU修改值强行同步回外圈物理主存(DDR)，防止iDMA搬走空壳旧数。
对齐铁律;以上两者的硬件清剿单位都是64字节，因此被操作的变量(如_mem1)必须使用ALIGNDCACHE(64字节对齐)，否则必然发生邻居变量被误伤的
数据踩踏。
## 2.4 port
TCM Port：外部访问VQ7 TCM内存，比如DSP启动是，把固件灌入IRAM中，把ISP图像数据送进DRAM中
ID Port：VQ7用指针读写一个外部的、零散的寄存器
DMA Port：VQ7配置一个大块的传输通道(Channe1)，并等待 DMA 中断，走的是 DMA Port，用于DTCM访间外部存储
ID Port占用VQ7 CPU资源 DMA不占
**CPU访问L1D和访问TCM两者互斥**
## 2.5 boot
![[file-20260602155247628.png]]
启动含义:
通过atol工具将x路径下的xx.bin download到DSP核的存储中(-d涉及到三个存储iram，dtr，psra，因为指令ACPU发出，所有这里的地址也是ACPU映射地址)，并且指示DSP的PC指针从x地址开始(-b)

NPD:Normal Power Down(标准掉电/正常掉电)

典型的多核固件下载流程通常是:
NPD进入idle模式：
{Halt (挂起副核)}，{Download (下载·bin 到副核内存)},  {(复位/启动副核)}
由于在复位后使能了Debug，副核虽然复位释放了，但它会因为调试模式生效而处于 Halt mode(挂起状态)，不会去执行 flash里可能存在的残余旧代码。此时，运行 Linux 的主核才能安全地通过总线，把新的.bin 固件下载到副核的私有内存中。
Reset vector：(复位向量)就是CPU 刚上电或发生复位(Reset)时，硬件规定必须去执行的第一条指令的内存地址，或者是存放该地址的指针。
## 2.6 reset
复位管辖          用途                                  例子
Dreset             DSP core                          TCM port
Breset             DSP作为master                 DSP使用iDMA访问外部
Preset             DSP作为master                 ACPU访问DSP IRAM

DSP_SC是冷启动，得通过上下电才能复位
实验现象:通过A55去访问IRAM，复位breset,可以正常访问，复位d/访问失败，目前推测b/preset应该和master/slave有关

devmem 0x02024b04 32 0xb0400000  解复位DSP  b/d/p reset
devmem 0x11005030  读DSP_SC寄存器   预期能看到PfaultError
![[file-20260602160747946.png]]

## 2.7 iDMA
idma搬运数据DRAM和DDR之间，在DRAM采用ping-pong buffer设计
idma_log_handler(idmaLogHander);  这是一个注册/设置函数（类似于系统的配置接口）。它的目的是告诉 iDMA 驱动：一旦未来发生了某种日志或错误事件，请去调用我传给你的这个工具。
DECLARE_PS(); 通常是一个宏定义（Macro），用于在函数的最开始声明一个用于保存处理器状态/中断状态的局部变量。
![[file-20260602161749970.png]]
图解
![[file-20260602163515221.png]]
参数
![[file-20260602163023558.png]]

2D 每行地址  地址=src+ (nrows_idex) * src_pitch
3D 每帧地址  地址=src+ (nrows_idx* src_pitch) + (ntiles_idx * src_tile_pitch)

max_pif_req = 32 总线控流、防堵塞的数量限制参数，表示：允许同时挂在总线上的最大在途请求数量  
Pif: process interface 处理器接口

idma_max_block_t  最大允许 PIF 请求块大小

![[file-20260602165529630.png|521]]


![[file-20260604110207383.png]]






# 3. 验证
## 3.1 启动验证
iram启动（默认方式）
atool download -m dsp -f /system/fw/dsp_idram_sram_fw_0.bin -t 2 -b 0x200000 -d 0x46001000 -c 0

ddr启动
atool download -m dsp -f /system/fw/dsp_idram_sram_fw_0.bin -t 1 -b 0x46000000 -d 0x46000000 -c 0

psram启动
atool download -m dsp -f /system/fw/dsp_idram_sram_fw_0.bin -t 1 -b 0x30000000 -d 0x30000000 -c 0

echo on > /sys/devices/platform/11005000.vdsp_core/power/control
dsp_control enable_wfi 0 0  


## 3.2 reset验证
DRESET测试：A55下
![[file-20260602160411666.png]]

devmem 0x11c00000 32 0x123  往DSP_IRAM写错值
devmem 0x02024014 32 0xb1234567  CRG_KEY
devmem 0x02024b00 32 0xb0400000  在ACPU子系统下复位DSP  b/d/p re