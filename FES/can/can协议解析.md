### 函数签名

c

static rt_bool_t parse_can_frame(struct rt_can_msg *can_msg, stim_msg_t *msg)

|参数|类型|含义|
|---|---|---|
|`can_msg`|输入|CAN 驱动读出来的原始帧（包含 ID、8 字节 data 等）|
|`msg`|输出|翻译后的服务层消息（调用者提前分配好内存）|
|返回值|`rt_bool_t`|`RT_TRUE` = 翻译成功，可以投递给服务层；`RT_FALSE` = 不需要投递（无效帧/组合帧未就绪）|

---

### 第一关：门卫检查

c

if (can_msg->id != CAN_ID_STIM_CMD)   // CAN_ID_STIM_CMD = 0x100

    return RT_FALSE;

CAN 总线上可能跑着各种各样的帧（别的设备、别的协议）。我们只认 **ID = 0x100** 的帧，其他统统不理。虽然硬件过滤器已经在做这个筛选了，但软件层再过滤一次更保险（**防御性编程**）。

---

### 第二关：取出数据部分

c

rt_uint8_t *d = can_msg->data;

CAN 帧的载荷是一个 **8 字节数组** `data[0]~data[7]`。这里用指针 `d` 只是为了后面写起来方便，`d[0]` 就等价于 `can_msg->data[0]`。

---

### 第三关：根据 `d[0]`（命令码）分支处理

`d[0]` 是我们协议规定的**命令码**，不同的值代表不同的操作：

#### 简单命令（1 帧搞定）

**START (0x01) 和 STOP (0x02)**：

c

case CAN_CMD_START:          // d[0] = 0x01

    msg->cmd = STIM_CMD_START;

    return RT_TRUE;          // 翻译完了，直接返回

这两个命令没有参数，只需要告诉服务层"开始"或"停止"，所以填好 `msg->cmd` 就完事了。

上位机发送的帧长这样：

ID=0x100, Data: [01] [00] [00] [00] [00] [00] [00] [00]

                 ↑

                命令码

---

#### 带参数的命令（1 帧搞定）

**SET_AMP (0x03)**：

c

case CAN_CMD_SET_AMP:

    msg->cmd = STIM_CMD_SET_AMP;

    msg->param.amp.pos_uA = (d[1] << 8) | d[2];   // 高字节在前

    msg->param.amp.neg_uA = (d[3] << 8) | d[4];

    return RT_TRUE;

上位机发送的帧：

Data: [03] [C3] [50] [C3] [50] [00] [00] [00]

       ↑    ↑────↑    ↑────↑

      命令  pos_uA   neg_uA

![](vscode-file://vscode-app/e:/Antigravity/resources/app/extensions/theme-symbols/src/icons/files/c.svg)

(d[1] << 8) | d[2] 这个操作是**大端拼接**：

d[1] = 0xC3 → 左移8位 → 0xC300

d[2] = 0x50 →                   

按位或 → 0xC350 = 50000 (十进制)

所以 `pos_uA = 50000`，就是 50mA。

---

#### 组合命令（需要 2 帧拼接才能搞定）

这是整个函数最复杂的部分。

**SET_TIME (0x04) — 第一帧**：

c

case CAN_CMD_SET_TIME:

    s_time_cache.freq_hz      = (d[1] << 8) | d[2];   // 频率

    s_time_cache.pos_width_us = (d[3] << 8) | d[4];   // 正脉宽

    s_time_cache.neg_width_us = (d[5] << 8) | d[6];   // 负脉宽

    s_time_cache.rx_tick      = rt_tick_get();          // 记下时间戳

    s_time_cache.pending      = RT_TRUE;                // 标记"我在等第2帧"

    return RT_FALSE;  // ← 注意！返回 FALSE，不投递！

上位机发送的帧：

Data: [04] [00] [32] [00] [C8] [00] [C8] [00]

       ↑    ↑────↑    ↑────↑    ↑────↑

      命令  freq=50   pos=200   neg=200

收到这帧后，函数返回 `RT_FALSE`。外面的调用者（

![](vscode-file://vscode-app/e:/Antigravity/resources/app/extensions/theme-symbols/src/icons/files/c.svg)

can_rx_thread_entry）看到 FALSE，就**不会**调用 

![](vscode-file://vscode-app/e:/Antigravity/resources/app/extensions/theme-symbols/src/icons/files/c.svg)

stim_service_send_cmd()。相当于这帧被"吃掉了"，暂存在 `s_time_cache` 里等待伙伴帧。

---

**SET_TIME_EXT (0x05) — 第二帧**：

c

case CAN_CMD_SET_TIME_EXT:

    if (s_time_cache.pending)    // 第1帧到了吗？

    {

        // 超时检测：第1帧是不是太久之前的？

        if ((rt_tick_get() - s_time_cache.rx_tick) > rt_tick_from_millisecond(50))

        {

            rt_kprintf("[CAN] TIME_EXT timeout, dropping\n");

            s_time_cache.pending = RT_FALSE;

            return RT_FALSE;     // 过期了，两帧都丢

        }

        // 没过期！拼接完整的消息

        msg->cmd = STIM_CMD_SET_TIME;

        msg->param.time.freq_hz      = s_time_cache.freq_hz;      // 从缓存取

        msg->param.time.pos_width_us = s_time_cache.pos_width_us;  // 从缓存取

        msg->param.time.neg_width_us = s_time_cache.neg_width_us;  // 从缓存取

        msg->param.time.dead_time_us = (d[1] << 8) | d[2];        // 从本帧取

        s_time_cache.pending = RT_FALSE;   // 清掉标记

        return RT_TRUE;  // ← 拼接完成！可以投递了

    }

    // 没有第1帧就直接来了第2帧？异常！

    rt_kprintf("[CAN] TIME_EXT without TIME\n");

    return RT_FALSE;

上位机发送的帧：

Data: [05] [00] [32] [00] [00] [00] [00] [00]

       ↑    ↑────↑

      命令  dead=50

**整个拼接过程的时序：**

时间线:  0ms          5ms                    55ms+

         │            │                       │

     收到 0x04     收到 0x05               如果此时才收到 0x05

     缓存数据      拼接成功! ✅             超时丢弃! ❌ (>50ms)

     pending=TRUE  返回 TRUE               打印 timeout

     返回 FALSE    投入服务层               返回 FALSE

---

### 其他简单命令

c

case CAN_CMD_CH_SET:                    // 0x10 设置单个电极

    msg->cmd = STIM_CMD_ELECTRODE_SET;

    msg->param.electrode.channel = d[1]; // 通道号 (0~31)

    msg->param.electrode.state   = d[2]; // 0=断开 1=闭合

    return RT_TRUE;

case CAN_CMD_CH_MASK:                   // 0x11 按掩码批量设置

    msg->cmd = STIM_CMD_ELECTRODE_MASK;

    msg->param.electrode_mask.chip_idx = d[1];            // 芯片索引

    msg->param.electrode_mask.mask     = (d[2] << 8) | d[3]; // 16位掩码

    msg->param.electrode_mask.state    = d[4];            // 0=断开 1=闭合

    return RT_TRUE;

---

### 总结

![](vscode-file://vscode-app/e:/Antigravity/resources/app/extensions/theme-symbols/src/icons/files/c.svg)

parse_can_frame 本质上就是一个**协议解码器**：

输入: 8 字节的原始 CAN 数据 (d[0]~d[7])

输出: 结构化的 stim_msg_t 消息

核心逻辑:

  1. d[0] 决定命令类型

  2. d[1]~d[7] 按大端序拼接成实际参数

  3. 特殊情况：SET_TIME 需要两帧才能拼完整

  4. 返回 TRUE = 消息完整可以投递, FALSE = 忽略或等待