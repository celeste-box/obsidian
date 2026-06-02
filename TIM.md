adb push sv_gtest /system/bin

ls /system/bin/ -al  
    
chmod +x /system/bin/sv_gtest

atool clock_show | grep timer   查看输出时钟



sv_gtest --gtest_filter=tim.TIM_REG_WRITE_READ

//sv_gtest --gtest_filter=tim.TIM_CMD_SW_RESET

sv_gtest --gtest_filter=tim.TIM_REG_CLR

sv_gtest --gtest_filter=tim.TIM_REG_INTR_CLR2

sv_gtest --gtest_filter=tim.TIM_REG_INTR_CLR

sv_gtest --gtest_filter=tim.TIM_REG_INTR_MANUAL

sv_gtest --gtest_filter=tim.TIM_REG_INTR_MASK

sv_gtest --gtest_filter=tim.TIM_REG_INTR_RAW_SET

sv_gtest --gtest_filter=tim.TIM_REG_STATE_GEN_00

sv_gtest --gtest_filter=tim.TIM_REG_STATE_GEN_00a     

sv_gtest --gtest_filter=tim.TIM_REG_STATE_GEN_00b  10

sv_gtest --gtest_filter=tim.TIM_REG_STATE_GEN_00c

sv_gtest --gtest_filter=tim.TIM_REG_STATE_GEN_00d

sv_gtest --gtest_filter=tim.TIM_REG_STATE_GEN_00e

sv_gtest --gtest_filter=tim.TIM_REG_STATE_GEN_00f

sv_gtest --gtest_filter=tim.TIM_REG_STATE_GEN_00g

sv_gtest --gtest_filter=tim.TIM_REG_STATE_GEN_00h

sv_gtest --gtest_filter=tim.TIM_REG_STATE_GEN_01

sv_gtest --gtest_filter=tim.TIM_REG_STATE_GEN_01a       

sv_gtest --gtest_filter=tim.TIM_REG_STATE_GEN_01b   20

sv_gtest --gtest_filter=tim.TIM_REG_STATE_GEN_02

sv_gtest --gtest_filter=tim.TIM_REG_STATE_GEN_02a

sv_gtest --gtest_filter=tim.TIM_REG_STATE_GEN_02b

sv_gtest --gtest_filter=tim.TIM_REG_STATE_GEN_03

sv_gtest --gtest_filter=tim.TIM_REG_STATE_GEN_03a

sv_gtest --gtest_filter=tim.TIM_REG_STATE_GEN_03b

sv_gtest --gtest_filter=tim.TIM_REG_STATE_GEN_03c

sv_gtest --gtest_filter=tim.TIM_REG_STATE_GEN_03d

sv_gtest --gtest_filter=tim.TIM_REG_STATE_GEN_04     

sv_gtest --gtest_filter=tim.TIM_REG_STATE_GEN_04a   30  

sv_gtest --gtest_filter=tim.TIM_REG_STATE_GEN_04b

sv_gtest --gtest_filter=tim.TIM_REG_STATE_GEN_04c

sv_gtest --gtest_filter=tim.TIM_REG_STATE_GEN_05

sv_gtest --gtest_filter=tim.TIM_REG_STATE_GEN_05a

sv_gtest --gtest_filter=tim.TIM_REG_STATE_GEN_06

sv_gtest --gtest_filter=tim.TIM_REG_PRESETN_CAM_RST_N

sv_gtest --gtest_filter=tim.TIM_WAVE_GEN_BROADCAST

sv_gtest --gtest_filter=tim.TIM_WAVE_EXT_BROADCAST_00a

sv_gtest --gtest_filter=tim.TIM_WAVE_EXT_BROADCAST_01    38

sv_gtest --gtest_filter=tim.DISABLED_TIM_WAVE_EXT_BROADCAST_00 --gtest_also_run_disabled_tests   

tim  clk  38.4M  26ns

TIM.WAVE.RESET.HW
sv_gtest --gtest_filter=tim.TIM_REG_STATE_GEN_00
sv_gtest --gtest_filter=tim.TIM_CMD_HW_RESET   IO 默认弱上拉，是高电平
sv_gtest --gtest_filter=tim.TIM_CMD_SW_RESET
sv_gtest --gtest_filter=tim.TIM_REG_STATE_GEN_00e



sv_gtest --gtest_filter=tim.TIM_CMD_HW_RESET
sv_gtest --gtest_filter=tim.TIM_CMD_SET_IO_4OUTPUT_POLAR_HIGH
sv_gtest --gtest_filter=tim.TIM_CMD_REFRESH_GEN0_15FPS
sv_gtest --gtest_filter=tim.TIM_CMD_START_GEN0_ONLY
sv_gtest --gtest_filter=tim.TIM_CMD_REFRESH_GEN0_30FPS

sv_gtest --gtest_filter=tim.TIM_CMD_SET_IO_4OUTPUT_POLAR_LOW

sv_gtest --gtest_filter=tim.TIM_CMD_SET_IO_2OUTPUT_2INPUT_POLAR_LOW

sv_gtest --gtest_filter=tim.TIM_CMD_REFRESH_GEN0_30FPS_GEN1_1140FPS



sv_gtest --gtest_filter=tim.TIM_CMD_START_GEN0_GEN1

sv_gtest --gtest_filter=tim.TIM_CMD_REFRESH_GEN0_1140FPS_LL

sv_gtest --gtest_filter=tim.TIM_CMD_START_GEN0_ONLY

sv_gtest --gtest_filter=tim.TIM_CMD_END_GEN0_IMMEDIATELY

sv_gtest --gtest_filter=tim.TIM_CMD_READ_SYSCNT --gtest_repeat=220

sv_gtest --gtest_filter=tim.TIM_CMD_READ_INTR

sv_gtest --gtest_filter=tim.TIM_CMD_READ_STREAM_STATE

sv_gtest --gtest_filter=tim.TIM_CMD_END_GEN0_NEXT_FRAME

sv_gtest --gtest_filter=tim.TIM_REG_STATE_GEN_03







