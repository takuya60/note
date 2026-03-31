### fdcan配置
代码中fdcan配置为80MHz 
仲裁位500K 数据位5M
# 数据协议
详细见`stim_fdcan_protocol.h`
未开启过滤器 所有id的帧的命令都可以接收
`data[0]` = 命令码,  `data[1~N]` = 参数
## 1.FES控制
| 命令码 | 名称    | 数据格式 | 说明 |
| `0x01`  | START | `[0x01]` | 开始刺激 |
| `0x02`  | STOP   | `[0x02]` | 停止刺激 |
| `0x03`  | SET_AMP | `[0x03, posH, posL, negH, negL]` | 设置电流幅值 (单位 µA) |
| `0x04`  | SET_TIME | `[0x04, freqH, freqL, posH, posL, negH, negL, deadH, deadL]` | 设置频率/脉宽/死区  (单位 µs) |
`(这里的数据是u16，所以分成H和L)`

| `0x0F`  | RESET_FAULT | `[0x0F]` | 复位故障状态|
`(只有在进入过流保护之后才可调用)`

| `0x20`  | SET_ALL | `[0x20, freqH, freqL, posH, posL, negH, negL, deadH, deadL, ampPosH, ampPosL, ampNegH, ampNegL]` | 一帧批量下发全部参数 |
## 2.通道控制
| 命令码 | 名称 | 数据格式 | 说明 |
| `0x10` | CH_SET | `[0x10, channel, state]` | 设置单通道 (channel: 0~31, state: 0/1) |
| `0x11` | CH_MASK | `[0x11, maskB3, maskB2, maskB1, maskB0]` | 按32位掩码设置全部通道 |
| `0x12` | CH_CLEAR | `[0x12]` | 清除所有通道，无参数 |
| `0x16` | CH_PAIR | `[0x16, anode, cathode]` | 设置成对电极 |

### 使用示例
先打开通道
- 16 01
这里指打开通道1对应的两个电极，对应关系见`stim_electrode.c`
再开始刺激
- 01
设置参数
- 03 13 88 13 88
再关闭刺激
- 02

