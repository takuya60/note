# 嵌入式自持 Web 服务器架构部署计划 (AMP 架构)

## 🎯 背景与目标

将 RK3506 彻底打造成一台**全自动化、开机即用**的“自持式边缘 Web 服务器”。 项目打破了原有的“外部上位机接收”思路，将底层固件数据采集、高频 SPI 总线交互、内核驱动、UDP 中间件乃至 3D 姿态展示 Web App 全部内聚到单板运行。只要板子一上电连网，用户便可通过浏览器直接查看 1000Hz 原生物理姿态的渲染画面，并具备工业级后台驻留和崩溃异常恢复能力。

## ⚙️ 系统架构与模块划分

本系统共分为由低到高的 5 个核心支撑层：

### 1. 硬件采集与驱动层 (STM32 端)

- **I2C 传感器采集**：编写 
    
    ![](vscode-file://vscode-app/e:/Antigravity/resources/app/extensions/theme-symbols/src/icons/files/c.svg)
    
    mpu6050.c，并完成重度硬件防卡顿优化（由 100kHz 升级为 **400kHz I2C Fast Mode**），启用传感器内置 PLL 和 42Hz DLPF，输出绝对 1000Hz 的原生姿态数据。
- **自定义 32 字节架构**：构建严格对齐的 `amp_frame_t` 头文件规范，首部装载 `0x5AA5` Magic Number 并尾随 `CRC16` 校验和。
- **DMA 防死锁引擎**：SPI 发送中加入 `2ms` 的自限时“心跳复原”机制。如果内核主板失联，它能自动清除 DMA 挂起状态并制造出源源不断的引脚下降沿中断脉冲。

### 2. 内核事件响应层 (RK3506 Linux Kernel)

- **SPI 事件中断 (
    
    ![](vscode-file://vscode-app/e:/Antigravity/resources/app/extensions/theme-symbols/src/icons/files/c.svg)
    
    stm32_amp.c)**：注册 `stm32-amp` 设备驱动及其硬件 IRQ，一旦检测到下降沿中断信号，马上非阻塞调度 `spi_async` 读取 32 字节总线数据。
- **用户态管道池**：通过自旋无锁 `kfifo` 存放采集的数据帧，利用 `wait_queue` 无缝唤醒应用层进程执行读取。

### 3. 系统核心分发层 (

![](vscode-file://vscode-app/e:/Antigravity/resources/app/extensions/theme-symbols/src/icons/files/c.svg)

udp_sender.c)

- **超高优先级进程**：Linux 下强制解开 `LimitRTPRIO=99` 封印，让接收端处于 `SCHED_FIFO` 实时调度模式与特定 CPU 核心强绑定，规避系统垃圾回收波浪导致的掉帧。
- **本地 Loopback 网络互联**：获取数据后立刻向本机的 `127.0.0.1:8888` 极速抛出 UDP 包，彻底斩断对外部路由器的通信依赖与延迟损耗。

### 4. 业务应用与可视化层 (

![](vscode-file://vscode-app/e:/Antigravity/resources/app/extensions/theme-symbols/src/icons/files/python.svg)

server.py & 

![145](vscode-file://vscode-app/e:/Antigravity/resources/app/extensions/theme-symbols/src/icons/files/code-orange.svg)

index.html)

- **Python 多线程总线**：利用 Python 异步监听本地 UDP 端口（8888）。将收到包含 `<6h` 高浓缩比特指令解析后，分发向 WebSocket 通道。
- **Web App 引擎**：自身充任 HTTP 端提供静态服务器（8080）。用户连入网页时，使用 `three.js` 以高重绘率绘制六轴传感器传入的模型俯仰、偏航动态。
- **联机终测**：完成烧录，拔除调试与串口，实现板子通电后依靠以太网即可通过 `8080` 端口流畅访问交互画面及后台追踪日志。

## 🌐 多端接入与公网访问方案

### 1. 局域网访问 (最推荐/最快)

- **场景**：手机、平板、电脑连接到与板子相同的 Wi-Fi/路由器上。
- **操作**：直接在浏览器输入 `http://<板子IP>:8080` 即可。
- **优点**：1000Hz 数据在高带宽下响应极快，无需互联网。

### 2. 内网穿透/公网分享 (远程给别人看)

- **场景**：你在外面，或者想发个链接给千里之外的朋友看实时效果。
- **方案 A (简单)**：使用 **ZeroTier / Tailscale** 建立虚拟局域网（只要手机也装个 App 就能像在家一样访问）。
- **方案 B (专业)**：在板子上跑 **frp** 或 **nps** 客户端。这需要你在外网有一台带固定 IP 的云服务器作为“跳板”。
- **方案 C (低延迟)**：如果你的家庭宽带自带**公网 IP**，直接在路由器设置“端口转发 (Port Forwarding)”，将外部 8080 端口映射到板子的内网 IP:8080。

### 3. 网线直连 (针对校园网/无网络环境)

- **场景**：板子没有 Wi-Fi，且校园网插上去要认证，导致手机和电脑都搜不到它。
- **操作**：直接用一根网线，一头插板子，一头插在你的笔记本网口上。
- **配置**：通常需要你在电脑的网卡设置里，手动指定一个静态 IP（如 `192.168.1.100`），并确保板子的 IP 也在同网段（如 `192.168.1.10`）。
- **优点**：极度稳定，物理隔绝校园网干扰，全速传输。

### 3. 关于域名 (Domain)

- **结论**：**不需要，除非你想给它起个好听的名字。**
- 域名只是 IP 的“马甲”。如果用公网访问，直接输入 `http://你的公网IP:8080` 也能打开。如果你追求极致体验，可以几块钱买个 `.top` 或 `.xyz` 的域名。

### 4. 网线直连 (针对校园网/无网络环境)

- **场景**：板子没有 Wi-Fi，且校园网插上去要认证，导致手机和电脑都搜不到它。
- **操作**：直接用一根网线，一头插板子，一头插在你的笔记本网口上。
- **配置**：需要你在电脑的网卡设置里，手动指定一个静态 IP（如 `192.168.1.100`），并确保板子的 IP 也在同网段（如 `192.168.1.10`）。
- **优点**：极度稳定，物理隔绝校园网干扰，全速传输。

TIP

**校园网/公共 Wi-Fi 特别说明** 校园网通常开启了 **AP 隔离 (Client Isolation)**，这意味着手机和板子虽然连着同一个 Wi-Fi，却互相“看不见”。 **解决方案**：买一个最便宜的**迷你路由器（旅行路由器）**插在宿舍，或者用一台电脑开**移动热点**，让手机和板子都连这个“私有 Wi-Fi”，这样就绝对互通了。

TIP

**校园网/公共 Wi-Fi 特别说明** 校园网通常开启了 **AP 隔离 (Client Isolation)**，这意味着手机和板子虽然连着同一个 Wi-Fi，却互相“看不见”。 **解决方案**：买一个最便宜的**迷你路由器（旅行路由器）**插在宿舍，或者用一台电脑开**移动热点**，让手机和板子都连这个“私有 Wi-Fi”，这样就绝对互通了。

### 5. 工业级自动部署守护层 (

![](vscode-file://vscode-app/e:/Antigravity/resources/app/extensions/theme-symbols/src/icons/files/shell.svg)

deploy.sh & `systemd`)

- **标准化部署脚手架**：一键式将工程执行文件和 Web 根目录自动洗包拷贝到 `/opt/amp/` 系统驻留专用目录。
- **生命周期看门狗**：
    - [NEW] 追加 
        
        ![](vscode-file://vscode-app/e:/Antigravity/resources/app/extensions/theme-symbols/src/icons/files/document.svg)
        
        amp-udp.service 配置文件。作为基础网关。
    - [NEW] 追加 
        
        ![](vscode-file://vscode-app/e:/Antigravity/resources/app/extensions/theme-symbols/src/icons/files/document.svg)
        
        amp-web.service 配置文件。严格挂载在前置程序的依赖链后，跟随系统开机静默启动，出现错误时交替执行限秒自动无限拉起。

---

## ⚠️ User Review Required

IMPORTANT

**联调卡壳点与硬件约束**

1. 当前联调正卡在 **Linux 内核感知不到单片机中断** 的硬件验证阶段（`/proc/interrupts` 无增长）。
2. 必须由运维人员（用户端）拿着原理图物理走查核对 STM32 的 `PB0` 引脚连上的 RK3506 引脚是否**完全等同于**设备树（Device Tree dts）内定义的 `rockchip_gpio_irq 8` 以及相关的 IO-Bank（如 `GPIO1_B0` 等），同时验证电平标准逻辑。