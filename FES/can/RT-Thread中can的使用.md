## 第一步：初始化阶段（stim_can_init 中做的事）

### 1. 打开 CAN 设备并注册中断
```c
rt_device_open(s_can_dev, RT_DEVICE_FLAG_INT_TX | RT_DEVICE_FLAG_INT_RX);
//                        ^发送用中断                ^接收用中断
```
这一句做了两件事：
- 打开 CAN 硬件外设
- 告诉 RT-Thread：**我要用中断方式收发**（而不是轮询方式）

底层会自动帮你配好 STM32 的 NVIC 中断向量，使能 FDCAN 的接收中断。**你不需要自己写寄存器配置。**

### 2. 设置接收回调
```c
rt_device_set_rx_indicate(s_can_dev, can_rx_callback);
```
这句话的意思是："当 CAN 硬件收到数据时，请调用 can_rx_callback 这个函数。"

RT-Thread 内部的连接关系：

STM32 FDCAN 硬件收到帧
    │
    ▼
STM32 FDCAN 中断触发 (硬件自动)
    │
    ▼

RT-Thread CAN 驱动的中断处理函数 (框架代码，你看不到)
    │
    │ 发现你注册了回调
    ▼
调用你的 can_rx_callback()

### 3. 初始化信号量
```c
rt_sem_init(&s_rx_sem, "can_rx", 0, RT_IPC_FLAG_FIFO);
//                         ^初始值=0，表示"还没有数据"
```
信号量就是一个**计数器**：

- 初始值 = 0，意思是"现在没有待处理的数据"
- 每次 `rt_sem_release` → 计数器 **+1**
- 每次 `rt_sem_take` → 计数器 **-1**，如果已经是 0 就**阻塞等待**

---

## 第二步：运行时流程

假设外部设备发来一个 CAN 帧 `[ID=0x100, data={0x01}]`（启动刺激命令）：

### ① 硬件中断触发

CAN 总线上的电平变化被 STM32 FDCAN 外设捕获，硬件自动将帧数据存入接收 FIFO，并触发中断。**这一步完全由硬件完成，不需要你写任何代码。**

### ② 你的回调被调用
```c

static rt_err_t can_rx_callback(rt_device_t dev, rt_size_t size)
{
    rt_sem_release(&s_rx_sem);   // 信号量计数: 0 → 1
    return RT_EOK;
}
```
**`rt_sem_release(&s_rx_sem)` 干了什么？**

两件事：

1. 信号量计数器 +1（从 0 变成 1）
2. 如果有线程正在 `rt_sem_take` 等待这个信号量，**立刻唤醒它**

**为什么回调里只做这一件事？** 因为回调在**中断上下文**中执行，有严格规则：

- ❌ 不能调用可能阻塞的函数
- ❌ 不能做耗时操作（会影响其他中断）
- ✅ 只能做极轻量的事（发个信号量、设个标志位）

### ③ 接收线程被唤醒


```c

static void can_rx_thread_entry(void *param)

{
    while (1)
    {
        // 之前阻塞在这里 (信号量=0，线程睡着，不占CPU)
        rt_sem_take(&s_rx_sem, RT_WAITING_FOREVER);
        // 信号量变成1了！线程醒来，继续执行 ↓
        // take之后，信号量自动重新变成0
        // 从 CAN 硬件 FIFO 中读出帧数据 
        //在字符设备（如 CAN、UART）中，这个参数通常**没有实际意义**，一般固定传 `0`
        rt_device_read(s_can_dev, 0, &can_msg, sizeof(can_msg));
        
        // 解析 CAN 帧 → stim_msg_t
        parse_can_frame(&can_msg, &msg);
        
        // 投入服务层消息队列
        stim_service_send_cmd(&msg);
        // 回到循环顶部，再次 rt_sem_take，如果没新数据就又睡着
    }
}
```
**`rt_sem_take(&s_rx_sem, RT_WAITING_FOREVER)` 干了什么？**

- 检查信号量计数器
- 如果 > 0：计数器 -1，立即返回（不阻塞）
- 如果 = 0：**线程挂起**（睡着），直到有人调用 `rt_sem_release` 唤醒它
- `RT_WAITING_FOREVER` 表示"永远等，不超时"

### ④ 服务层执行命令
```c

// stim_service.c 的服务线程

rt_mq_recv(&s_cmd_mq, &msg, sizeof(msg), 50);

// 取到 STIM_CMD_START → handle_start() → 开始刺激
```
---

## 完整时序图

时间轴 →

外部设备:     ═══[发送CAN帧]═══

                      │

CAN硬件:              ├── 接收帧 → FIFO → 触发中断

                      │

中断回调:             ├── rt_sem_release() → 信号量 0→1

                      │

接收线程:   [阻塞等待]├── rt_sem_take()返回 → 读帧 → 解析 → send_cmd

                      │

服务线程:   [等待队列] ├── rt_mq_recv()返回 → handle_start() → 电刺激启动

                      │

整个过程耗时:    约 100~500微秒

**核心思想就是三个字：中断 → 信号量 → 线程**。这是 RTOS 中处理外部事件的标准模式。