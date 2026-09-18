# 1. 框图
![[file-20260729105535266.png]]
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
Cache Flush /Clean(回写):配合“DMA发送”。把L1D里的CPU修改值强行同步回外圈物理主存(DDR)，防止iDMA搬走空壳旧数。
对齐铁律;以上两者的硬件清剿单位都是64字节，因此被操作的变量(如_mem1)必须使用ALIGNDCACHE(64字节对齐)，否则必然发生邻居变量被误伤的
数据踩踏。

xthal_dcache_region_invalidate：cache Invalid
XT_MEMW：Xtensa 官方说明，`MEMW` 会对它前后的 load、store 和 cache operation 建立顺序，所以是保证执行顺序
xthal_dcache_region_writeback：Cache Flush
## 2.4 port
TCM Port：外部访问VQ7 TCM内存，比如DSP启动是，把固件灌入IRAM中，把ISP图像数据送进DRAM中
ID Port：VQ7用指针读写一个外部的、零散的寄存器
DMA Port：VQ7配置一个大块的传输通道(Channe1)，并等待 DMA 中断，走的是 DMA Port，用于DTCM访间外部存储
ID Port占用VQ7 CPU资源 DMA不占
**CPU访问L1D和访问TCM两者互斥**
## 2.5 boot -- atool
![[file-20260602155247628.png]]
启动含义:
通过atool工具将x路径下的xx.bin download到DSP核的存储中(-d涉及到三个存储iram，dtr，norflash，因为指令ACPU发出，所有这里的地址也是ACPU映射地址)，并且指示DSP的PC指针从x地址开始(-b)

NPD:Normal Power Down(标准掉电/正常掉电)

典型的多核固件下载流程通常是:
NPD进入idle模式：
{Halt (挂起副核)}，{Download (下载·bin 到副核内存)},  {(复位/启动副核)}
由于在复位后使能了Debug，副核虽然复位释放了，但它会因为调试模式生效而处于 Halt mode(挂起状态)，不会去执行 flash里可能存在的残余旧代码。此时，运行 Linux 的主核才能安全地通过总线，把新的.bin 固件下载到副核的私有内存中。
Reset vector：(复位向量)就是CPU 刚上电或发生复位(Reset)时，硬件规定必须去执行的第一条指令的内存地址，或者是存放该地址的指针。

atool运行在ACPU侧
    │
    ├─ -d：ACPU视角的目标地址
    │      将bin写入DSP IRAM/DTR/PSRAM等存储
    │
    └─ -b：写入DSP reset vector的启动PC
           这个值最终由DSP作为取指地址使用
    
boot流程
1. core1/2 ，先与DSP_NOC_NPD完成握手
2. 复位glb/b/d/p
3. 解复位glb
4. 配置dsp_sc，配置CTRL0和RESETVEC；必须在glb解复位之后
5. 解复位b/d/p
6. 加载固件
7. run core


## 2.6 reset
复位管辖          用途                                  例子
Dreset             DSP core                          TCM port
Breset             DSP作为master                 DSP使用iDMA访问外部
Preset             DSP作为master                 ACPU访问DSP IRAM    

已和设计确认：*breset是core reset.     dreset是复位debug（OCD)逻辑。  preset是复位APB口控制的逻辑*

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
··
![[file-20260602165529630.png|521]]

idma_buf_struct  一个数组就对应一个task
![[file-20260604155817666.png]]
![[file-20260604155857563.png]]

**idma task搬数流程**
- **`idma_init_task`**：初始化任务结构体，主要是重置 `next_add_desc` 指针，使其指向第 0 个描述符的起点（即准备好第一个“空格子”）。
    
- **`idma_add_2d_desc`**：按照 1D/2D/3D 硬件需要的格式填充描述符，填完后 **`next_add_desc` 指针自动向后推进**（注意：这里**不操作** `fixed-buffer`，因为这是 TASK 模式）。
    
- **`idma_schedule_task`**：更新并提交任务。在更新前**关闭中断**防止状态被打乱；若硬件忙，则通过修改前一个任务的描述符插入 `JUMP` 指令挂载到链表末尾；若硬件闲，则直接配置寄存器启动搬运。
    
- **`idma_hw_wait_all`**：让当前线程进入休眠，挂起等待，直到 DMA 搬运完毕触发中断，将线程唤醒，宣告工作完成。
## 2.8 VRF/ARF
矢量/标量寄存器
## 2.9 CBB/DBB
- **DBB (Data Backbone / Data Bus) —— 数据骨干网**
    
    - **职责**：数据平面。专门用来在内存（如 DRAM）和高性能外设（如 DMA 引擎、GPU、NPU、网卡）之间，**大批量地搬运数据**。
        
- **CBB (Control Backbone / Control Bus) —— 控制骨干网**
    
    - **职责**：控制平面。专门用来让 CPU 去读写各个外设的**配置寄存器（Registers）**，下达命令、修改时钟、重置状态等。

## 2.9 QMAN IPC
整体工作关系
发送端写 DDR ringbuffer
        ↓
通过 QMAN channel 通知对端
        ↓
QMAN 产生中断
        ↓
接收端读取 DDR ringbuffer

## 2.10 核启动
启动顺序：
本地 ResetVector
    ↓
配置当前 Core 的 multi-entry remap
    ↓
memw / isync
    ↓
跳转到 DDR 或 NORFLASH 中的 ResetHandler/正文
    ↓
C运行库初始化
    ↓
board_init()

stall_dsp(core);
config_dsp(core, boot_addr);
release_dsp(core);

ACPU加载所有FW段 
load_dsp_fw(...);

DSP仍处于runstall，没有取指 
config_dsp_remap(core);

 最后才让DSP开始取指 
run_dsp(core);



## 2.11 muti-entry remap
AXI/NoC 地址通常按“字节”寻址：每一个地址对应1 Byte。
|30 bit|`2^30`|1 GiB|
|32 bit|`2^32`|4 GiB|
|36 bit|`2^36`|64 GiB|
|40 bit|`2^40`|1 TiB|

典型配置顺序
 1. 先关闭该 entry 
2. 写 DSP 侧输入地址 
3. 写映射后的 system 地址 
4. 写 size/mask/attribute 
5. 最后使能 entry 
6. readback + memw，确保寄存器配置生效 
### 2.11.1 ID remap
原理：
DSP原始地址（32bit）
        ↓
    entry remap
        ↓
NoC输出地址（36bit）

entry_mask用于控制 `[29:12]` 中参与比较的高位 bit；再加上永远参与比较的 `[31:30]`，共同决定匹配窗口大小。
mask=4KB：比较 input 和 start 的全部高20bit    0x1000
mask=1GB：只比较 input 和 start 的最高2bit

规则：
1. 输入地址高 20bit 与 16 个 entry 的 start addr（20bit start addr）进行匹配，如果匹配成功则使用该 entry 的 target_addr 替换输入地址高位，输入地址低位不变；

2. 如果输入地址同时与多个 entry 匹配，则上报中断给 DSP，且用编号较大且匹配成功的 entry；

3. 若都没匹配成功，这时候 output_addr = {4'b0000, input_addr};【透传】

- 访问4GB以下地址：不配置remap，直接透传。
- 访问4GB以上DDR：选择一个4GB以下的DSP输入窗口，将其target配置成实际的36-bit DDR目标地址。
### 2.11.2 DMA remap
DMA remap主要解决DMA 36bit VA到36bit PA的重定位和隔离。
DDR视角被remap



## 2.12 BWC&BWL
- QoS/Regulator：验证实时流量在竞争下能够获得所需带宽。
- BWL：验证 BWC 发出限制请求后，指定的非实时流量被限制在带宽上限以内。

# 3 代码移植

dsp仓库
https://gerrit.imv.local/c/manifests/+/84914

cd ../../dsp/boards/im_a500/a500_fpga
source ../../../xtensa_env_setup.sh vdsp

xt-genldscripts -b ./ldconfig/core2_local/
rm -f ./ldconfig/core2_local/mpu_table.c
手动刷新xmm对应的ldscripts下的文件
## 3.1 整体代码
|目录|主要职责|新 feature 放置建议|
|---|---|---|
|`boards/im_a500/a500_fpga`|板级入口、每核配置、LSP、板级头文件|FPGA/板级差异|
|`boards/im_a500`|A500 BSP、设备配置、启动适配、portable|A500 SoC 相关适配|
|`driver/include`|驱动公共接口和数据结构|可复用驱动 API|
|`driver/src`|IPC、share buffer、device 等驱动实现|正式驱动功能|
|`app`|固件主程序、任务、模型和算法入口|产品功能、算法调度|
|`test/sv_test/common`|DSP core、XOS、IDMA、异常、指令功能测试|与具体 SoC 无关的 core 验证|
|`test/sv_test/im_a500/vdsp`|QMAN、IPCM、外部中断、地址空间等|DSP subsystem/A500 集成验证|
|`utils/cmd-parser`|测试命令注册和解析|通常不放具体测试|
|`include`、`lib`|算法头文件及预编译库|公共算法依赖|
|`host`|主机侧调用和接口|需要 PC/AP 配合的工具|
|`scripts`|固件打包和辅助脚本|构建、转换工具|

## 3.2dsp编译
整理链路
Kconfig/.config
  ↓
im.mk 选择核和板级目录
  ↓
build_link_file.sh 生成每核 LSP [LSP 是 Xtensa 的 Linker Support Package，中文可以理解为“链接器支持包”]
  ↓
a500_fpga/Makefile
  ↓
coreN_config_entry.mk
  ↓
vdsp.mk：VDSP 专属代码、算法 .mk、工具链参数
  ↓
common.mk：扫描源码、编译、链接、拆分、打包
  ↓
三个 dsp_idram_sram_fw_N.bin
### 3.2.1 顶层编译开关
im/build/cfg/dsp.cfg
        ↓ 生成
im/dsp/ImConfig
        ↓ Kconfig
im/boards/a500_fpga/.config
        ↓
im/dsp/im.mk

开关：
BUILD_IM_DSP
BUILD_IM_DSP_CORE1
BUILD_IM_DSP_CORE2
DSP_PROFILE="a500_fpga"
DSP0/1/2_USING_DDR_BASE/SIZE

im.mk 根据 DSP_PROFILE 拼出： IM_DSP_BOARD = dsp/boards/im_a500/a500_fpga
并分别调用：
make all CORE=0
make all CORE=1
make all CORE=2

### 3.2.2 生成链接脚本
im/boards/a500_fpga/config/ddr_layout  源头
        ↓
ddr_layout.h 和 .config
        ↓
im/dsp/build_link_file.sh
        ↓
coreN_local/memmap.xmm   各核修改
        ↓ xt-genldscripts
coreN_local/ldscripts/

### 3.2.3 Xtensa环境
xtensa_env_setup.sh 

### 3.2.4 板级Makefile入口
dsp/boards/im_a500/a500_fpga/Makefile
收到
make all CORE=N CONFIG_ENTRY=coreN_0_config_entry.mk
负责确定
BOARD_T = a500_fpga
TARGET  = A500-LOCAL
CORE    = N

### 3.2.5 每核功能确定
core0_0_config_entry.mk
core1_0_config_entry.mk
core2_0_config_entry.mk

### 3.2.6 VDSP专属配置
dsp/boards/im_a500/vdsp.mk

### 3.2.7 公共编译和链接规则
dsp/boards/im_a500/common.mk

### 3.2.8 最终链接和打包
所有 .c/.cpp
    ↓
每核独立 .o
    ↓
使用 coreN_local/ldscripts 链接
    ↓
dsp_idram_sram_fw_N
    ↓ objcopy
dsp_iram_fw_N.elf
dsp_dram_fw_N.elf
dsp_sram_fw_N.elf
    ↓ generate_bin_with_header.sh
dsp_idram_sram_fw_N.bin













## 3.3 dsp不同启动方式下的内存划分
dsp/boards/im_a500/a500_fpga/ldconfig



pkgs/im/hal/platform/dsp/config  ???



dsp/boards/im_a500/a500_fpga/ldconfig
core0：用于ddr启动
core0_local：用于iram启动
core0_nor_flash：用于nor_flash启动，待添加
所以上述其实是不同启动方式的视角
xthal_MPU_entry，在涉及到cache的时候需要，iram(core0_local)因为是TCM AXI 不涉及cache，纯在自家范围内


dsp/boards/im_a500/a500_fpga/ldconfig/core0/ldscripts 下的elf32xtensa.* 是根据memmap.xmm自动生成的
![[file-20260630154921709.png]]

![[file-20260630162622297.png]]
![[file-20260630162652182.png]]

boards/a500_fpga/.config   开关
![[file-20260704173008973.png]]
![[file-20260701105109612.png]]
![[file-20260701105724114.png]]

![[file-20260701111605988.png]]
build_link_file.sh
![[file-20260701161310850.png]]

![[file-20260701163457694.png]]

DSP_CONFIG_NAME 在这里赋值
![[file-20260702154036308.png]]

## 3.4 A500下的编译适配
common.mk
1. 去除heartbeat.c编译
![[file-20260901175318479.png]]

2. 去除call_stack整个文件夹下内容编译，但因为又关联到exc_handler链接和cmd_task.c，所以采用弱处理+构造空函数
![[file-20260901175425575.png]]

3.  xmm链接的Xtensa编译，一目录有问题需要改成VQ7_prod，二需要根据当前路径自动生成路径
![[file-20260901175724054.png]]

# 4. core 验证

command -v devmem
command -v busybox
busybox --list | grep '^devmem$'
busybox devmem 0x14c51054 32 6  
busybox devmem 0x14c51054 32 7


dmesg -n 1

busybox devmem 0x14c51054 32 0x6
busybox devmem 0x14c51054 32 0x7



adb push .\atool /system/bin/atool
adb shell chmod 755 /system/bin/atool
adb push .\dsp_idram_sram_fw_0.bin /system/fw/dsp_idram_sram_fw_0.bin
adb push .\dsp_idram_sram_fw_1.bin /system/fw/dsp_idram_sram_fw_1.bin
adb push .\dsp_idram_sram_fw_2.bin /system/fw/dsp_idram_sram_fw_2.bin


查看是否上电  busybox devmem 0x12033078 32          0x007C7FFF

crg  
写使能   
devmem 0x12034014 32 0x01234567
devmem 0x12034018 32 0x76543210
busybox devmem 0x12034874 32 

pmctrl
devmem 0x12033014 32 0x04168990
devmem 0x12033098 32 0x001C7fff
devmem 0x120330a8 32 0x001C7fff

devmem 0x120330a0 32
devmem 0x120330ac 32

## 4.1 启动验证
adb push dsp_idram_sram_fw_0.bin system/fw/dsp_idram_sram_fw_0. bin
iram启动（默认方式）
atool download -m dsp -f /system/fw/dsp_idram_sram_fw_0.bin -t 2 -b 0x200000 -d 0x46001000 -c 0
atool download -m dsp -f /system/fw/dsp_idram_sram_fw_1.bin -t 2 -b 0x200000 -d 0x56001000 -c 1
atool download -m dsp -f /system/fw/dsp_idram_sram_fw_2.bin -t 2 -b 0x200000 -d 0x76001000 -c 2

atool download -m dsp -f ./dsp_idram_sram_fw_0.bin -t 2 -b 0x200000 -d 0x46001000 -c 0
atool download -m dsp -f ./dsp_idram_sram_fw_0.bin -t 2 -b 0x200000 -c 0
atool download -m dsp -f ./dsp_idram_sram_fw_1.bin -t 2 -b 0x200000 -c 1
atool download -m dsp -f ./dsp_idram_sram_fw_2.bin -t 2 -b 0x200000 -c 2

atool download -m dsp -f /system/fw/dsp_idram_sram_fw_0.bin -t 2 -b 0x200000 -c 0
atool download -m dsp -f /system/fw/dsp_idram_sram_fw_1.bin -t 2 -b 0x200000 -c 1
atool download -m dsp -f /system/fw/dsp_idram_sram_fw_2.bin -t 2 -b 0x200000 -c 2



atool download -m dsp -f /system/fw/dsp_idram_sram_fw_0.bin -t 2 -b 0x200000 -d 0x46001000 -c 0
ddr启动
atool download -m dsp -f /system/fw/dsp_idram_sram_fw_0.bin -t 1 -b 0x46000000 -d 0x46000000 -c 0
atool download -m dsp -f ./dsp_idram_sram_fw_0.bin -t 1 -b 0x46000000 -d 0x46000000 -c 0

norflash启动
atool download -m dsp -f /system/fw/dsp_idram_sram_fw_0.bin -t 1 -b 0x30000000 -d 0x30000000 -c 0

echo on > /sys/devices/platform/11005000.vdsp_core/power/control
dsp_control enable_wfi 0 0  


## 4.2 reset验证
DRESET测试：A55下
![[file-20260602160411666.png]]

devmem 0x11c00000 32 0x123  往DSP_IRAM写错值
devmem 0x02024014 32 0xb1234567  CRG_KEY
devmem 0x02024b00 32 0xb0400000  在ACPU子系统下复位DSP  b/d/p re


![[file-20260903172345511.png]]

## 4.3 指令验证
1. DSP.FORMAT.001  测试  SIMD INT
resnet50_test
![[file-20260911120228597.png]]


im/dsp/app/insta/resnet50_model.c注册模型
im/dsp/app/insta/lib/libResNet50_main.a  模型lib
im/dsp/test/sv_test/common/resnet50_test.c   测试代码，配置模型参数

![[file-20260910165800864.png]]
分析：
resnet50输入输出放到DDR上
resnet50模型内部算子需要dsp local ram且需求很大，因此需要将固件原本放在 local DRAM 的静态段搬到 DDR，从而把 local DRAM 腾给模型算子。
在ddr启动的基础上
原先：
![[file-20260911112401491.png]]

修改：将dram.rodata/data/bss
![[file-20260911112340831.png]]

![[file-20260911112605345.png]]

2. DSP.FORMAT.002  测试   SIMD FP
由resnet50_test覆盖

3. DSP.PERFORMANCE.001.001  测试  SIMD/VRF INT最大性能（MACs）
由resnet50_test覆盖

4. DSP.PERFORMANCE.001.002
由resnet50_test覆盖

5. DSP.PERFORMANCE.001.003
int32_mac_test
![[file-20260827151411537.png]]

6. DSP.PERFORMANCE.002.001  测试  SIMD/VRF FP最大性能（并行度）
fmac_test
![[file-20260827145319385.png]]

7. DSP.PERFORMANCE.002.002
由fmac_test覆盖

8. DSP.PERFORMANCE.002.003
由fmac_test覆盖

10. DSP.PERFORMANCE.003  LdST最大性能
由resnet50_test覆盖

11. DSP.PERFORMANCE.004   SPU最大性能：10级流水线
coremark_test
得分：3333/921.6M=3.62
![[file-20260828214106424.png]]

问题：
![[file-20260828180231507.png]]

分析：
运行时间不足10s导致的报错
![[file-20260828192740237.png]]
![[file-20260828192752234.png]]
![[file-20260831105853909.png]]

解析：
`ITERATIONS == 0` 的自动校准逻辑本来应该让正式测试超过 10 秒，但是会出现下列这种情况：计时器发生了回绕，导致性能数据无效
![[file-20260830165005843.png]]
理论解决方法，未尝试
将XT_RSR_CCOUNT换成xos_get_system_cycles
![[file-20260831112230499.png]]


12. DSP.PERFORMANCE.005　ISA最大性能：5-way VLIW
coproc_test
![[file-20260827151631966.png]]

从dump文件看coproc_test包括了5-way VLIW指令，执行coproc_test
![[file-20260828142354568.png]]

![[file-20260827151151914.png]]

Boolean指令验证
![[file-20260828143731939.png]]

13. DSP.PERFORMANCE.006　　低功耗
测试leakage power

## 4.4 debug验证
1. IP.DSP.RST.005     复位
1.reset设置
2.先单步运行几步，关注pc值有无变化
3.点击reset，观察复位后的pc值
![[file-20260915173243100.png]]

![[file-20260916172152379.png]]



2. IP.DSP.IF_DBG.001.001    配置JTAG连接DSP，通过ocd进行等同GDB的debug
![[file-20260916172400859.png]]

3. IP.DSP.IF_DBG.001.002     打断点
![[file-20260915181136404.png]]

![[file-20260916190044555.png]]
![[file-20260917114700560.png]]

4.  IP.DSP.IF_DBG.001.003     通过efuse关闭debug权限
1 在uboot启动时通过efuse关闭debug权限
2 通过atool加载dsp
3 启动xt-ocd报错
![[file-20260916105809732.png]]

![[file-20260916193136074.png]]

![[file-20260917115811465.png]]

5. DSP_SUBSYS.DFX.003
在IP.DSP.IF_DBG.001.003  基础上，efuse关闭debug权限后，可通过DCU将debug权限重新打开
![[file-20260916143011141.png]]
![[file-20260916110911090.png]]

![[file-20260916193340540.png]]

![[file-20260917115914044.png]]

6. DSP.DFX.001.003    performeance monitor功能测试
debug连接后，运行prfmtr_test
![[file-20260916150400154.png]]

# 5 subsys验证
## 5.1 复位验证     
==**带有npd模块的正常复位流程就是先npd握手再配置复位信号**==
reset core/noc，必须在core idle的时候
1. DSP_Subsys.RST.001      b/d/p/glb复位
由boot启动覆盖

2. DSP_Subsys.RST.003      noc复位
noc_resetn   
A55: 
devmem 0x12034014 32 0x1234567
devmem 0x12034b74 32 
devmem 0x16400000 32
devmem 0x12034b6c 32 0x1
devmem 0x12034b74 32 
devmem 0x16400000 32   系统挂死
![[file-20260902172912175.png]]

noc_core0_resetn
A55: 先给core1/2上电
devmem 0x12034014 32 0x1234567
devmem 0x12034b74 32 
devmem 0x1640c000 32  core0
devmem 0x1640d000 32  core1
devmem 0x1640e000 32  core2
devmem 0x12034b6c 32 0x8
devmem 0x12034b74 32 
devmem 0x1640d000 32  core1
devmem 0x1640e000 32  core2
devmem 0x1640c000 32  core0  系统挂死
![[file-20260902180841395.png]]


复位core1/2之前需要先做npd握手
noc_core1_resetn
A55: 先给core1/2上电
devmem 0x12034014 32 0x1234567
devmem 0x12034b74 32 
devmem 0x1640c000 32  core0
devmem 0x1640d000 32  core1
devmem 0x1640e000 32  core2
devmem 0x12033268 32 0x10001    npd握手
devmem 0x12033290 32
devmem 0x12034b6c 32  0x10
devmem 0x12034b74 32 
devmem 0x1640c000 32  core0  正常访问
devmem 0x1640e000 32  core2  正常访问
devmem 0x1640d000 32  core1  返回err
![[file-20260908165815493.png]]

noc_core2_resetn
A55: 先给core1/2上电
devmem 0x12034014 32 0x1234567
devmem 0x12034b74 32 
devmem 0x1640c000 32  core0
devmem 0x1640d000 32  core1
devmem 0x1640e000 32  core2
devmem 0x12033268 32 0x20002   npd握手
devmem 0x12033290 32
devmem 0x12034b6c 32  0x20
devmem 0x12034b74 32 
devmem 0x1640c000 32  core0      正常访问
devmem 0x1640d000 32  core1      正常访问
devmem 0x1640e000 32  core2      返回err
![[file-20260908172127730.png]]

noc_div4_resetn   
A55: 
devmem 0x12034014 32 0x1234567
devmem 0x12034b74 32 
devmem 0x16400000 32
devmem 0x12034b6c 32 0x4
devmem 0x12034b74 32 
devmem 0x16400000 32   系统挂死
![[file-20260902193641806.png]]

noc_div2_resetn   
A55: 
devmem 0x12034014 32 0x1234567
devmem 0x12034b74 32 
devmem 0x16400000 32
devmem 0x12034b6c 32 0x2
devmem 0x12034b74 32 
devmem 0x16400000 32   系统挂死
![[file-20260902194334895.png]]
## 5.2 idma验证
1. DSP_Subsys.IF.001.001
   由其他idma用例覆盖
2. DSP_Subsys.IF.001.002      idma可以访问宽度为128bit的数据
idma可以正常搬运数据即认为测试成功，由其他idma用例覆盖

3. DSP_Subsys.IF.001.003      idma可以访问16g地址范围的数据
idma_16g_test
![[file-20260903165556645.png]]

4. DSP_Subsys.IF.001.004       idma可以对不同channel完成数据搬运
idma_2ch_test
![[file-20260903163627216.png]]

5. DSP_Subsys.IF.001.005       idma可以按照一个task 一个task完成任务
idma_task_test
![[file-20260905104605643.png]]
![[file-20260904170717154.png]]

6. DSP_Subsys.IF.001.006      idma可以完成DDR和DRAM之间数据搬运
idma_2D_test   由DSP_Subsys.IF.001.014  2D覆盖

7. DSP_Subsys.IF.001.007       idma内jump命令可以修改数据读取或写入顺序
idma_task_test   由DSP_Subsys.IF.001.005 覆盖

8. DSP_Subsys.IF.001.008      可以得到idma状态
idma_PingPongBuf_test
![[file-20260904154220579.png]]
![[file-20260903165814590.png]]

9. DSP_Subsys.IF.001.009       DMA产生中断
idma_task_test   由DSP_Subsys.IF.001.005 覆盖
红色为idma错误判断
10. DSP_Subsys.IF.001.010      DMA握手测试
sv无法直接验证，认为数据拷贝正确，src和dst一致则认为测试通过，由其他idma用例覆盖

11. DSP_Subsys.IF.001.011      idma的buffermode，idma支持buffmode和task mode
idma_buffmode_36bit_test
![[file-20260903170004779.png]]

12. DSP_Subsys.IF.001.012          idma的cache一致性验证
idma_16g_test  
![[file-20260904180718219.png]]
关闭cache后
![[file-20260904172453584.png]]
![[file-20260904181427006.png]]

13. DSP_Subsys.IF.001.013      idma pingpong buffer
idma_PingPongBuf_test，由DSP_Subsys.IF.001.008覆盖

14. DSP_Subsys.IF.001.014      idma 1D/2D/3D数据搬运测试
1D   idma_16g_test     由DSP_Subsys.IF.001.003 覆盖
2D   idma_2D_test
![[file-20260904142123347.png]]

3D   idma_3D_test
![[file-20260903170532519.png]]
![[file-20260903170547843.png]]

![[file-20260903170711695.png]]
![[file-20260903170724009.png]]

![[file-20260903170757478.png]]
![[file-20260903170739199.png]]

15. DSP_Subsys.IF.001.015      idma pause和resume
idma_pause_resume_test
![[file-20260903171208314.png]]

16. DSP_Subsys.IF.001.016    idma step测试
idma_pause_resume_test   DSP_Subsys.IF.001.015覆盖

17. DSP_Subsys.IF.001.017   idma 和 xdma功能验证
让idma循环搬运，再此期间运行xdma


18. DSP_Subsys.IF.004     支持dsp0~2通过ID port和dma port互相访问对方的I/DRAM
core_communication
![[file-20260908150552548.png]]



## 5.3通路验证
1. DSP_Subsys.IF.002          AXI4 slave 接口
DSP_Subsys.IF.002.001    外部arm可以将指令和数据写入TCM
A55侧
core0:
devmem 0x16500000 32 0xaa
devmem 0x16500000
devmem 0x16580000 32 0xaa
devmem 0x16580000

core1:
devmem 0x16600000 32 0xbb
devmem 0x16600000
devmem 0x16680000 32 0xbb
devmem 0x16680000

core2:
devmem 0x16700000 32 0xcc
devmem 0x16700000
devmem 0x16780000 32 0xcc
devmem 0x16780000
![[file-20260903113023453.png]]

DSP_Subsys.IF.002.002   core将指令和数据写入各自的TCM
DSP侧
devmem 0x00200000 32 0xab
devmem 0x00200000 32 
devmem 0x00280000 32 0xab
devmem 0x00280000
![[file-20260903113745096.png]]

DSP_Subsys.IF.002.003  任意两核可以互相访问
DSP侧
core0：
devmem 0x16600000 32 0x1e
devmem 0x16600000
devmem 0x16680000 32 0x1e
devmem 0x16680000
devmem 0x16700000 32 0x2e
devmem 0x16700000
devmem 0x16780000 32 0x2e
devmem 0x16780000
![[file-20260903113937546.png]]
core1:
devmem 0x16500000 32 0xdd
devmem 0x16500000
devmem 0x16580000 32 0xdd
devmem 0x16580000
devmem 0x16700000 32 0x2d
devmem 0x16700000
devmem 0x16780000 32 0x2d
devmem 0x16780000
![[file-20260903114141133.png]]
core2:
devmem 0x16500000 32 0xcc
devmem 0x16500000
devmem 0x16580000 32 0xcc
devmem 0x16580000
devmem 0x16600000 32 0x1c
devmem 0x16600000
devmem 0x16680000 32 0x1c
devmem 0x16680000
![[file-20260903114243231.png]]


2. DSP_Subsys.IF.003     AXI master
DSP_Subsys.IF.003.001   外部master访问TCM/外设
TCM访问由DSP_Subsys.IF.002.001 覆盖
外设访问：
A55侧
devmem 0x16401004 32 0x44
devmem 0x16401004 32 
devmem 0x16402004 32 0x44
devmem 0x16402004 32 
devmem 0x16403004 32 0x44
devmem 0x16403004 32 
![[file-20260903114503989.png]]

DSP_Subsys.IF.003.002   访问DDR
DSP侧
core0
devmem 0x40000000 32 0x1234
devmem 0x40000000 32

core1
devmem 0x40000000 32 0x2345
devmem 0x40000000 32

core2
devmem 0x40000000 32 0x3456
devmem 0x40000000 32
![[file-20260903114721189.png]]

DSP_Subsys.IF.003.003   访问SHRAM
DSP侧
core0
devmem 0x1F000000 32 0x1234
devmem 0x1F000000 32

core1
devmem 0x1F000000 32 0x2345
devmem 0x1F000000 32

core2
devmem 0x1F000000 32 0x3456
devmem 0x1F000000 32
![[file-20260903114906737.png]]

## 5.4 系统验证
1. DSP_Subsys.SS_SC.002   wdt暂停测试
wdt0:
devmem 0x16400010 32 0xACCE55
devmem 0x16401004 32 0xbb
devmem 0x16401000 32 0x3
devmem 0x16401008 32  读数
devmem 0x16400030 32 0xFFAC01
devmem 0x16401008 32  
![[file-20260903150424556.png]]

wdt1:
devmem 0x16400010 32 0xACCE55
devmem 0x16402004 32 0xbb
devmem 0x16402000 32 0x3
devmem 0x16402008 32  读数
devmem 0x16400030 32 0xFFAC02
devmem 0x16402008 32  
![[file-20260903151430067.png]]

wdt2:
devmem 0x16400010 32 0xACCE55
devmem 0x16403004 32 0xbb
devmem 0x16403000 32 0x3
devmem 0x16400030 32 0xFFAC04
devmem 0x16403008 32  
![[file-20260903151440925.png]]

2. DSP_Subsys.SS_SC.003   SS_SC REV寄存器
devmem 0x16400010 32 0xACCE55
devmem 0x16400070 32 0xaa
devmem 0x16400074 32 0xaa
devmem 0x16400078 32 0xaa
devmem 0x1640007c 32 0xaa
devmem 0x16400080 32 0xaa
devmem 0x16400084 32 0xaa
devmem 0x16400088 32 0xaa
devmem 0x1640008c 32 0xaa
![[file-20260903145627179.png]]

3. DSP_Subsys.SS_SC.003   SS_SC 支持配置CORE ID
core0：
devmem 0x16400010 32 0xACCE55
devmem 0x16400034 32
devmem 0x16400034 32 0x4
devmem 0x16400034 32

core1：
devmem 0x16400010 32 0xACCE55
devmem 0x16400038 32
devmem 0x16400038 32 0x4
devmem 0x16400038 32

core2：
devmem 0x16400010 32 0xACCE55
devmem 0x1640003c 32
devmem 0x1640003c 32 0x4
devmem 0x1640003c 32
![[file-20260903145919744.png]]


4. DSP_Subsys.SC.003
修改错误启动地址  将0x200000改成0x123
A55侧
devmem 0x1640C028 32
![[file-20260903173339155.png]]
devmem 0x1640D028 32
![[file-20260903173601985.png]]cd..
devmem 0x1640E028 32
![[file-20260903173714886.png]]

5. DSP_Subsys.SC.005     SC支持DMAtrigin信号触发，以及DMAtrigout的状态观察
idma_2D_trig_in_test
idma_2D_trig_out_test
![[file-20260905141843226.png]]

## 5.5 ID remap验证
1. DSP_Subsys.SC.006.001     ID remap功能验证   
由其余ID remap用例覆盖

2. DSP_Subsys.SC.006.002    ID remap优先级测试 ，当entry范围重叠时，访问的是entry编号高的remap地址
id_remap_overlap
在id_remap_1g基础上，给2个entry写不同的值，A55回读确认是否是entry编号高的remap地址
![[file-20260905154509164.png]]
![[file-20260905171334717.png]]
![[file-20260905182038307.png]]

3. DSP_Subsys.SC.006.003   访问entry未覆盖地址会直接访问原地址
id_remap_uncover
在id_remap_4k基础上，dsp写一个未覆盖的地址，并去回读
![[file-20260905154709126.png]]
![[file-20260905171426581.png]]
![[file-20260905181648229.png]]


4. DSP_Subsys.SC.006.004    ID remap enable/disable功能验
id_remap_4k
id_remap_enable_disable
在id_remap_4k基础上，先由DSP在回读映射前地址，关闭后在回读一次
![[file-20260905161927455.png]]
![[file-20260905171600461.png]]
![[file-20260905181611786.png]]

5. DSP_Subsys.SC.006.005    ID remap step验证  4K/1G
id_remap_4k
由A55回读remap后的地址值是否符合预期
![[file-20260905162354378.png]]
![[file-20260905171713787.png]]
![[file-20260905181316756.png]]

id_remap_1g
由A55回读remap后的地址值是否符合预期
![[file-20260905162518828.png]]
![[file-20260905171800953.png]]
![[file-20260905181134799.png]]


## 5.6 dma remap
1. DSP_Subsys.SC.006.006
由其余dma remap用例覆盖


2. DSP_Subsys.SC.006.007    优先级测试，entry编号高的优先级高
dma_remap_overlap
![[file-20260907172938207.png]]

3. DSP_Subsys.SC.006.008   未覆盖地址会直接访问原地址
dma_remap_uncover
![[file-20260907173033645.png]]

4. DSP_Subsys.SC.006.009   enable/disable功能
dma_remap_enable_disable
![[file-20260907173118595.png]]


5. DSP_Subsys.SC.006.010   step验证，128K/4M
dma_remap_128k
![[file-20260907172840385.png]]


dma_remap_4m
![[file-20260907173247804.png]]


6. DSP_Subsys.SC.006.011   场景验证，拼接SHRAM和DDR
dma_remap_shram_ddr_concat
![[file-20260907173629331.png]]




# 6. debug环境搭建

Xplorer
  ↓ GDB协议，TCP 20000
xt-ocd
  ↓
J-Link驱动
  ↓ JTAG
Xtensa JTAG TAP
  ↓
XDM
  ↓
DSP Core、寄存器和内存

XDM 负责执行这些操作：
- 暂停、继续和单步 DSP
- 产生 Debug Interrupt
- 读取/修改 DSP 寄存器
- 访问 DSP 内存
- 设置硬件断点
- 获取 DSP 的复位、供电和调试状态
- 支持 Trace/TRAX

- `xt-ocd`：电脑上的服务程序，把 GDB 命令转换成 XDM/JTAG 操作。
- J-Link：发送 JTAG 电信号的探针。
- JTAG TAP：XDM 对外的 JTAG入口，直连模式下 IR width 通常是 5。
- XDM：真正控制 DSP 调试状态的硬件模块。
- `OCDDebugStall`：SoC 控制寄存器发给 DSP/XDM 的外部暂停请求，不等于开启 XDM。
- `PdebugEnable`：主要用于 PDebug/Trace 信息输出，不是 XDM 总使能开关。

## 6.1安装Xplorer
![[file-20260911165500060.png]]
安装信息：
INSTALLDIR = D:\work\VDSP\xtensa
XX_NAME = Xplorer-9.0.20
XT_VER = RI-2022.10
XT_TOOLS_NAME = XtensaTools_RI_2022_10_win32.tgz
PROD_VER = 9.0.20
Congratulations !!  You have finished installing Xplorer-9.0.20
Please review the following message log to make sure of the success of the installation.

=========================================================================================
Post-installation step #1
....INSTALL XtensaTools
FOUND WHERE_UTILS_PATH="D:\work\VDSP\xtensa\Xplorer-9.0.20\eclipse\plugins\other.xide.external.utils_9.1.2.3000"
....INSTALL XOS Document Plugin
FOUND WHERE_XOS_ZIP="D:\work\VDSP\xtensa\XtDevTools\install\tools\RI-2022.10-win32\XtensaTools\doc\xos-3.01.zip"
READY_TO_RUN_XOS_CMD=D:\work\VDSP\xtensa/Xplorer-9.0.20/eclipse/jre/bin/java.exe -cp D:\work\VDSP\xtensa\Xplorer-9.0.20\eclipse\plugins\other.xide.external.utils_9.1.2.3000/utils.jar;D:\work\VDSP\xtensa\Xplorer-9.0.20\eclipse\plugins\other.xide.external.libutils_9.0.20.3000.jar  other.xide.external.utils.io.Unpack D:\work\VDSP\xtensa\XtDevTools\install\tools\RI-2022.10-win32\XtensaTools\doc\xos-3.01.zip D:\work\VDSP\xtensa/Xplorer-9.0.20/eclipse/dropins D:\work\VDSP\xtensa\Xplorer-9.0.20\eclipse\plugins\other.xide.external.utils_9.1.2.3000
....INSTALL XIPC Document Plugin
FOUND WHERE_XIPC_ZIP="D:\work\VDSP\xtensa\XtDevTools\install\tools\RI-2022.10-win32\XtensaTools\doc\xipc.zip"
READY_TO_RUN_XIPC_CMD=D:\work\VDSP\xtensa/Xplorer-9.0.20/eclipse/jre/bin/java.exe -cp D:\work\VDSP\xtensa\Xplorer-9.0.20\eclipse\plugins\other.xide.external.utils_9.1.2.3000/utils.jar;D:\work\VDSP\xtensa\Xplorer-9.0.20\eclipse\plugins\other.xide.external.libutils_9.0.20.3000.jar other.xide.external.utils.io.Unpack D:\work\VDSP\xtensa\XtDevTools\install\tools\RI-2022.10-win32\XtensaTools\doc\xipc.zip D:\work\VDSP\xtensa/Xplorer-9.0.20/eclipse/dropins D:\work\VDSP\xtensa\Xplorer-9.0.20\eclipse\plugins\other.xide.external.utils_9.1.2.3000

Post-installation step #2
....UNPACK CONFIG
....INSTALL CONFIG
....INITIALIZE Xtensa Xplorer
.
Unpack Xtensa configs from directory D:/work/VDSP/xtensa/XtDevTools/downloads/RI-2022.10/builds

=========================================================================
UNPACK CONFIG XRC_FusionF1_All_cache
INSTALL CONFIG XRC_FusionF1_All_cache

The installation process is now complete.
INSTALL CONFIG RESULT :: 0
READY TO Set Xtensa Registry for XRC_FusionF1_All_cache
Shortcut Link created C:\ProgramData\Microsoft\Windows\Start Menu\Programs\RI-2022.10\XRC_FusionF1_All_cache CMD Shell.lnk
Shortcut Link created C:\ProgramData\Microsoft\Windows\Start Menu\Programs\RI-2022.10\XRC_FusionF1_All_cache ISA Ref..lnk
=========================================================================
UNPACK CONFIG hifi3_ss_spfpu_7
INSTALL CONFIG hifi3_ss_spfpu_7

The installation process is now complete.
INSTALL CONFIG RESULT :: 0
READY TO Set Xtensa Registry for hifi3_ss_spfpu_7
Shortcut Link created C:\ProgramData\Microsoft\Windows\Start Menu\Programs\RI-2022.10\hifi3_ss_spfpu_7 CMD Shell.lnk
Shortcut Link created C:\ProgramData\Microsoft\Windows\Start Menu\Programs\RI-2022.10\hifi3_ss_spfpu_7 ISA Ref..lnk
=========================================================================
UNPACK CONFIG hifi3z_ss_spfpu_7
INSTALL CONFIG hifi3z_ss_spfpu_7

The installation process is now complete.
INSTALL CONFIG RESULT :: 0
READY TO Set Xtensa Registry for hifi3z_ss_spfpu_7
Shortcut Link created C:\ProgramData\Microsoft\Windows\Start Menu\Programs\RI-2022.10\hifi3z_ss_spfpu_7 CMD Shell.lnk
Shortcut Link created C:\ProgramData\Microsoft\Windows\Start Menu\Programs\RI-2022.10\hifi3z_ss_spfpu_7 ISA Ref..lnk
=========================================================================
UNPACK CONFIG hifi4_ss_spfpu_7
INSTALL CONFIG hifi4_ss_spfpu_7

The installation process is now complete.

Non-interactive mode
  Xtensa Tools location:     D:/work/VDSP/xtensa/XtDevTools/install/tools/RI-2022.10-win32/XtensaTools
  Xtensa Core registry:      D:/work/VDSP/xtensa/XtDevTools/install/tools/RI-2022.10-win32/XtensaTools/config
  Register as default:       no
  Replace same-named config: yes

INSTALL CONFIG RESULT :: 0
READY TO Set Xtensa Registry for hifi4_ss_spfpu_7
Shortcut Link created C:\ProgramData\Microsoft\Windows\Start Menu\Programs\RI-2022.10\hifi4_ss_spfpu_7 CMD Shell.lnk
Shortcut Link created C:\ProgramData\Microsoft\Windows\Start Menu\Programs\RI-2022.10\hifi4_ss_spfpu_7 ISA Ref..lnk
=========================================================================
UNPACK CONFIG sample_config
INSTALL CONFIG sample_config

The installation process is now complete.

Non-interactive mode
  Xtensa Tools location:     D:/work/VDSP/xtensa/XtDevTools/install/tools/RI-2022.10-win32/XtensaTools
  Xtensa Core registry:      D:/work/VDSP/xtensa/XtDevTools/install/tools/RI-2022.10-win32/XtensaTools/config
  Register as default:       no
  Replace same-named config: yes

INSTALL CONFIG RESULT :: 0
READY TO Set Xtensa Registry for sample_config
Shortcut Link created C:\ProgramData\Microsoft\Windows\Start Menu\Programs\RI-2022.10\sample_config CMD Shell.lnk
Shortcut Link created C:\ProgramData\Microsoft\Windows\Start Menu\Programs\RI-2022.10\sample_config ISA Ref..lnk
=========================================================================
UNPACK CONFIG sample_controller
INSTALL CONFIG sample_controller

The installation process is now complete.
INSTALL CONFIG RESULT :: 0
READY TO Set Xtensa Registry for sample_controller
Shortcut Link created C:\ProgramData\Microsoft\Windows\Start Menu\Programs\RI-2022.10\sample_controller CMD Shell.lnk
Shortcut Link created C:\ProgramData\Microsoft\Windows\Start Menu\Programs\RI-2022.10\sample_controller ISA Ref..lnk
=========================================================================
UNPACK CONFIG sample_flix
INSTALL CONFIG sample_flix

The installation process is now complete.
INSTALL CONFIG RESULT :: 0
READY TO Set Xtensa Registry for sample_flix
Shortcut Link created C:\ProgramData\Microsoft\Windows\Start Menu\Programs\RI-2022.10\sample_flix CMD Shell.lnk
Shortcut Link created C:\ProgramData\Microsoft\Windows\Start Menu\Programs\RI-2022.10\sample_flix ISA Ref..lnk
=========================================================================
UNPACK CONFIG tie_dev1
INSTALL CONFIG tie_dev1

The installation process is now complete.

Non-interactive mode
  Xtensa Tools location:     D:/work/VDSP/xtensa/XtDevTools/install/tools/RI-2022.10-win32/XtensaTools
  Xtensa Core registry:      D:/work/VDSP/xtensa/XtDevTools/install/tools/RI-2022.10-win32/XtensaTools/config
  Register as default:       no
  Replace same-named config: yes

INSTALL CONFIG RESULT :: 0
READY TO Set Xtensa Registry for tie_dev1
Shortcut Link created C:\ProgramData\Microsoft\Windows\Start Menu\Programs\RI-2022.10\tie_dev1 CMD Shell.lnk
Shortcut Link created C:\ProgramData\Microsoft\Windows\Start Menu\Programs\RI-2022.10\tie_dev1 ISA Ref..lnk
=========================================================================
UNPACK CONFIG tie_dev2
INSTALL CONFIG tie_dev2

The installation process is now complete.

Non-interactive mode
  Xtensa Tools location:     D:/work/VDSP/xtensa/XtDevTools/install/tools/RI-2022.10-win32/XtensaTools
  Xtensa Core registry:      D:/work/VDSP/xtensa/XtDevTools/install/tools/RI-2022.10-win32/XtensaTools/config
  Register as default:       no
  Replace same-named config: yes

INSTALL CONFIG RESULT :: 0
READY TO Set Xtensa Registry for tie_dev2
Shortcut Link created C:\ProgramData\Microsoft\Windows\Start Menu\Programs\RI-2022.10\tie_dev2 CMD Shell.lnk
Shortcut Link created C:\ProgramData\Microsoft\Windows\Start Menu\Programs\RI-2022.10\tie_dev2 ISA Ref..lnk
=========================================================================
    Setting up Xtensa Xplorer configuration
    Initialize Xtensa Xplorer configuration cache
    Initialize Xtensa Xplorer RESULT:0


## 6.2 License安装
windows，此电脑右键/属性，右边高级系统设置，环境变量，系统变量，新建变量‘XTENSAD_LICENSE_FILE’，值为'27050@10.198.37.44'
![[file-20260914111853816.png]]
![[file-20260914112039341.png]]

## 6.3VQ7配置
![[file-20260911171759726.png]]


![[file-20260911171833407.png]]


![[file-20260911172031814.png]]

## 6.4安装jlink驱动
![[file-20260911174741316.png]]

## 6.5安装xt-ocd
安装包位置
![[file-20260911174902197.png]]

安装位置
![[file-20260911174930717.png]]
## 6.6 xt-ocd配置
右击属性，将下面的命令复制到快捷方式的目标中
"C:\Program Files (x86)\Tensilica\Xtensa OCD Daemon 14.10\xt-ocd.exe" "-dTD=80" "-locdlog.txt" "-cC:\Program Files (x86)\Tensilica\Xtensa OCD Daemon 14.10\DSP_jlink_ocd_debug_topology_jtag_core0.xml"

"C:\Program Files (x86)\Tensilica\Xtensa OCD Daemon 14.10\xt-ocd.exe" "-dTD=80" "-locdlog.txt" "-cC:\Program Files (x86)\Tensilica\Xtensa OCD Daemon 14.10\DSP_jlink_ocd_debug_topology_jtag_core1.xml"



硬件上需要先连接jlink并加载dsp固件，然后运行xt-ocd.exe，下图是正常运行结果
![[file-20260915120441054.png]]

注意点：
DSP_jlink_ocd_debug_topology_dap_core0.xml是通过apb debug
DSP_jlink_ocd_debug_topology_jtag_core0.xml是通过jlink debug

起始地址必须保证得有写权限
![[file-20260914143030966.png]]


运行J-Link Commander，获取序列号
![[file-20260914143148351.png]]
这里的序列号需要与J-Link Commander保持一致
![[file-20260914143355077.png]]

## 6.7 调试步骤
设置源码
![[file-20260915145816322.png]]

![[file-20260915145930497.png]]

![[file-20260915150029713.png]]
Compilation path是elf文件中的编译路径，Local file system path是对应的本机映射路径
Compilation path:
/home/celestechen/work/T2515/0901/im/dsp

Local file system path:
X:\work\T2515\0901\im\dsp

![[file-20260915150201356.png]]

![[file-20260915150233655.png]]

![[file-20260915150347782.png]]

具体步骤
执行atool download
devmem 0x12030010 32 0xACCE55      
devmem 0x12030050 32 0x301        core0  top_sc选择JTAG，否则XDM TAP IR  width识别不到5
devmem 0x12030010 32 0xACCE55      
devmem 0x12030050 32 0x303        core1 top_sc选择JTAG，否则XDM TAP IR  width识别不到5
devmem 0x12030010 32 0xACCE55      
devmem 0x12030050 32 0x403        core2  top_sc选择JTAG，否则XDM TAP IR  width识别不到5
启动 xt-ocd，使用 jtag_core0.xml

![[file-20260915115753085.png]]


![[file-20260916171648529.png]]


先关闭 Xplorer，然后使用同一个 workspace 清理界面状态：
"D:\vdsp\Xplorer-9.0.20\eclipse\eclipse.exe" -clean -clearPersistedState -data "D:\vdsp\Xplorer-clean-20260915"







# 其他
  
板子修改：  
1. mount -o remount,rw /dev/mmcblk0p6 /system  
2. 修改文件/system/bin/load_driver 注释掉dsp驱动开机自启
boards/a400_udp/system/bin/load_driver:DSP_MODULES="dsp"

![](file:///C:/Users/celestechen/Documents/WXWork/1688857586784348/Cache/Image/2026-06/企业微信截图_17816906567841.png)  



![[file-20260916185647735.png]]
