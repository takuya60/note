## 🚀 项目说明文档：基于 AMP 架构的超低延迟 6DOF 空间交互系统

### 1. 项目概述 (Project Overview)

本项目旨在设计并实现一套具备微秒级硬件时间戳对齐与极低端到端延迟的 6 自由度（6DOF）空间姿态追踪系统。系统摒弃了传统的单片机直连 Wi-Fi 方案，采用 **RK3506 (Linux) + STM32 (FreeRTOS)** 的非对称多处理 (AMP) 架构，彻底剥离硬实时传感器采样与复杂网络协议栈的资源抢占问题。

本项目深度涉及 Linux 内核驱动开发（SPI DMA、中断下半部、输入子系统）、并发无锁队列设计以及 Linux 网络栈调优，是一套极具工业落地价值的高频边缘网关架构原型。

### 2. 核心硬件拓扑 (Hardware Topology)

- **硬实时协处理器 (Slave):** STM32F1/F4 系列。负责 1000Hz 频率下 MPU6050/9250 的 I2C 轮询、姿态解算及微秒级时间戳生成。
    
- **边缘计算主控 (Master):** RK3506 开发板。负责运行定制化 Linux 系统，承载自研 SPI 平台驱动与网络分发任务。
    
- **高速通信总线:** 20MHz SPI 结合硬件 DMA 引脚（外加一根独立的 GPIO 用于触发 Linux 硬件中断）。
    
- **无线数据链路:** 插入 RK3506 USB Host 的 5GHz USB Wi-Fi 网卡，建立点对点 (P2P) 热点直连。
    

### 3. 软件架构与内核子系统设计 (Software Architecture)

**A. STM32 固件层 (硬实时域)**

- **时钟基准:** 启用高精度硬件定时器 (TIM)，为每一次 IMU 采样生成全局递增的微秒级时间戳 (Timestamp)。
    
- **姿态解算:** 移植 Madgwick AHRS 互补滤波算法，将原始加速度与角速度转换为平滑的四元数 (Quaternion) 和欧拉角。
    
- **通信发送:** 通过 SPI DMA 链表模式，在不占用单片机主循环 CPU 的情况下，将打包好的姿态帧推流给 RK3506。
    

**B. RK3506 Linux 内核层 (深水区核心)**

- **设备树 (Device Tree):** 编写自定义 DTS 节点，声明 SPI 外设属性、最大时钟频率及绑定的 GPIO 中断引脚。
    
- **中断上下半部 (Top/Bottom Half):** * **硬中断:** 极速响应 STM32 拉低的 GPIO 信号，清零中断标志位后立刻返回。
    
    - **工作队列 (Workqueue):** 唤醒下半部任务，配置 Linux SPI 子系统的 DMA 控制器读取姿态数据流。
        
- **并发与同步机制:** 在内核中开辟 `kfifo`（无锁环形缓冲区）存放高速流入的 SPI 数据。引入 `wait_queue`（等待队列），实现阻塞型 I/O，极大降低 CPU 轮询开销。
    
- **Input 子系统集成:** 将驱动注册为标准的 `evdev` 输入设备，向内核上报 `EV_ABS` (绝对姿态) 事件，在 `/dev/input/` 下生成标准设备节点。
    

**C. RK3506 Linux 用户态与网络层**

- **零拷贝穿透 (可选增强):** 提供 `mmap` 接口，允许核心应用直接映射内核姿态数据页。
    
- **网络栈极致调优:** 使用 C 语言编写 UDP 发送服务。调用 `sendmmsg` 批量处理系统调用；通过 `setsockopt` 注入 `IPTOS_LOWDELAY` (Type of Service) 提高内核网络队列的转发优先级。
    
- **实时进程调度:** 通过 `pthread_setaffinity_np` 进行 CPU 绑核，并配置 `SCHED_FIFO` 实时调度策略，确保发送进程不被系统后台抢占。
    

---

### 4. 项目开发里程碑计划 (Development Milestones)

|**阶段**|**核心任务**|**技术输出/检验标准**|**预计耗时**|
|---|---|---|---|
|Phase 1<br><br>  <br><br>硬件底层打通|1. 跑通 STM32 I2C 读取 MPU6050<br><br>  <br><br>2. 移植四元数解算算法<br><br>  <br><br>3. STM32 配置 SPI Slave 与 DMA 发送|串口能持续打印 1000Hz 不卡顿的平滑姿态数据和微秒时间戳。|优先完成|
|Phase 2<br><br>  <br><br>内核驱动框架|1. 编写 RK3506 侧设备树 (DTS) 节点<br><br>  <br><br>2. 编写 Linux SPI 平台驱动框架<br><br>  <br><br>3. 实现 GPIO 中断捕获与打印|`dmesg` 能看到内核成功响应 STM32 的硬件中断触发信号。|核心难点|
|Phase 3<br><br>  <br><br>高阶内核机制|1. 驱动内引入工作队列处理 SPI 读取<br><br>  <br><br>2. 实现 `kfifo` 环形队列与阻塞 `read`<br><br>  <br><br>3. 注册 Linux Input 子系统|`cat /dev/input/eventX` 能看到疯狂刷新的乱码（代表事件流打通）。|核心难点|
|Phase 4<br><br>  <br><br>网络与呈现|1. 编写 C 语言 UDP `sendmmsg` 客户端<br><br>  <br><br>2. 落实绑核与 `SCHED_FIFO` 调度<br><br>  <br><br>3. 电脑端编写简易 3D 接收界面|电脑端 3D 模型实现“指哪打哪”的零延迟视觉同步跟随。|成果展示|

---

### 5. 简历亮点提炼 (Resume Bullet Points)

_此部分可直接用于未来的求职简历中_

- **异构系统架构设计：** 主导设计 RK3506 (Linux) + STM32 (RTOS) 的 AMP 边缘计算架构。彻底解耦高频传感器硬实时采样与复杂网络协议栈，消除单核 MCU 方案中网络中断导致的 1000Hz 采样时钟抖动 (Jitter)。
    
- **Linux 核心驱动开发：** 弃用低效的用户态轮询，基于设备树独立开发 SPI 平台驱动。采用 Top-Half/Bottom-Half 中断架构结合内核 DMA，解决高频中断风暴；引入 `kfifo` 无锁队列与 `wait_queue` 阻塞 I/O，将驱动 CPU 占用率控制在 5% 以下。
    
- **标准子系统集成：** 深入剖析 Linux Input 子系统，将 6DOF 传感器数据抽象为标准 `EV_ABS` 事件流上报，无缝兼容标准 Linux 输入框架，极大提升代码可移植性与规范性。
    
- **网络栈底层调优：** 针对海量微小数据包传输痛点，使用 C 语言调用 `sendmmsg` 优化内核上下文切换开销；配置 `IPTOS_LOWDELAY` 提高网络队列优先级，结合 `SCHED_FIFO` 绑核调度，实现微秒级时间戳数据的零延迟 UDP 传输。