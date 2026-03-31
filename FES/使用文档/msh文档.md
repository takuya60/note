#### 使用示例
```c
msh /> stim init                # 初始化(如果没自动初始化)
msh /> stim pair 1              # 第1路通道
msh /> stim time 100 200 200 50 # 100Hz, 正负脉宽200us, 死区50us
msh /> stim amp 500 500         # 正负幅值均设为 500uA
msh /> stim start               # 启动输出
msh /> stim status              # 查看当前输出状态和电压/电流反馈
msh /> stim stop                # 停止输出
```
### 刺激控制与参数设置

- **启动/停止**: `stim start` / `stim stop`
- **复位故障**: `stim reset`
- **设置幅值**: `stim amp <正向幅值> <反向幅值>`  
- **设置时间**: `stim time <频率> <正向脉宽> <反向脉宽> <死区>`
- **查看状态**: `stim status` (显示当前系统状态、参数及电极闭合情况)
### 电极与通道管理

- **通道设置**: `stim pair <1~16>`
   (快速切换预定义的电极对，通道对对应的电极在`stim_electrode.c`查看)
- **物理对**: `stim ppair <pos_ch> <neg_ch>` 
   (直接控制 0~31 号物理电极)
- **掩码控制**: `stim chmask <hex_32>` 
  (例：`stim chmask FFFFFFFF` 开启所有电极)
- **全部清除**: `stim chclear`
- **分时复用 (TDM)**:
    - 配置: `stim tdm config <ch1> [ch2...ch4]`
    - 控制: `stim tdm start` / `stim tdm stop` / `stim tdm status`
###  ADC采集

- **初始化/控制**: `stim adc <init|start|stop>`
- **查看特征值**: `stim adc show [ch]` (查看峰值、物理均值等)
- **打印原始值**: `stim adc dump [ch] [n]` (打印前 n 个采样点)

### 测试专用

- **CAN测试**: `stim cantest` (持续循环发送 CAN 测试帧)
- **SPI测试**: `stim spitest` (每秒 1000 次 SPI 指令压测)