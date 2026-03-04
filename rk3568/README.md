# ⚡ High-Performance Medical Electrical Stimulation System
# 高性能医疗电刺激控制系统

![Qt6|75](https://img.shields.io/badge/Qt-6.5+-41cd52?style=flat&logo=qt)
![Platform](https://img.shields.io/badge/Platform-RK3568%20%7C%20Linux-blue)
![Architecture](https://img.shields.io/badge/Arch-Heterogeneous%20Dual--Core-orange)
![License](https://img.shields.io/badge/license-MIT-green)

> 一个基于 **RK3568 (Linux)** 与 **Cortex-M0** 双核异构架构的专业医疗电刺激设备控制系统。采用 Qt6/QML 构建现代化 UI，通过 SPI 实现微秒级实时控制与波形回传。

## 📸 项目演示 (Screenshots)

|          参数设置 (Parameter Setup)           |           实时监控 (Real-time Monitor)            |
| :---------------------------------------: | :-------------------------------------------: |
| ![ParamPage](./doc/images/param_page.png) | ![MonitorPage](./doc/images/monitor_page.png) |
|          *支持 us/ms 级参数精细调节与波形预览*          |            *Canvas 高性能实时波形绘制与能量统计*            |

---

## 🏗 系统架构 (System Architecture)

本项目采用 **上位机 + 下位机** 的异构设计，确保了 UI 的流畅性与脉冲输出的硬实时性。

* **Host (上位机)**: Rockchip RK3568 (Arm64)
    * 运行嵌入式 Linux 系统。
    * 基于 **Qt6.x + QML** 开发，负责业务逻辑、数据可视化、用户交互。
    * 通过 `spidev` 驱动与下位机通讯。
* **Slave (下位机)**: Cortex-M0 单片机
    * 负责 PWM 波形发生、ADC 同步采集、硬件急停保护。
    * 作为 SPI Slave 响应上位机指令。

## ✨ 核心功能 (Key Features)

* **微秒级精准控制**: 支持 `us` (微秒) 和 `ms` (毫秒) 级脉宽调节，最大支持 2000us 微调。
* **实时波形监控**: 基于 QML `Canvas` 实现 50Hz+ 的高帧率电压/电流波形绘制。
* **多维数据分析**: 实时计算 **瞬时功率 (mW)**、**累计能量 (J)**、**负载阻抗 (Ω)**。
* **智能安全保护**:
    * 电极脱落检测 (Open Circuit Detection)。
    * 过流/过压保护。
    * 通信超时自动急停。
* **现代化 UI 设计**: 采用 Glassmorphism (毛玻璃) 风格，支持深色模式，触控友好。

---

## 🔌 硬件连接 (Hardware Connection)

通信接口采用 **SPI (Mode 0, 8-bit)**。请确保 **RK3568 (Master)** 与 **M0 (Slave)** 共地。

| 信号名称 | RK3568 (40-Pin Header) | M0 Pin | 描述 |
| :--- | :--- | :--- | :--- |
| **MOSI** | Pin 19 (SPI3_MOSI_M1) | PA7 | 主机发送 -> 从机接收 |
| **MISO** | Pin 21 (SPI3_MISO_M1) | PA6 | 从机发送 -> 主机接收 |
| **CLK** | Pin 23 (SPI3_CLK_M1) | PA5 | 时钟信号 |
| **CS** | Pin 24 (SPI3_CS0_M1) | PA4 | 片选信号 (低电平有效) |
| **GND** | Pin 6/9/14... | GND | **必须连接！** |

---

## 🛠️ 快速开始 (Getting Started)

### 1. 环境依赖 (Prerequisites)
* **硬件**: RK3568 开发板 (如 Orange Pi 3B, Firefly Station P2 等)。
* **系统**: Debian 11 / Ubuntu 20.04 (Linux Kernel 5.10+)。
* **软件**: Qt 6.2 或更高版本 (交叉编译环境)。

### 2. 开启 SPI 设备树 (Device Tree)
在 RK3568 上，需开启 `spi3-m1` 节点。
* **方法 A (推荐)**: 使用 `rsetup` 或 `orangepi-config` 进入 `Hardware` 设置，勾选 `spi3-m1-cs0`。
* **方法 B (手动)**: 编辑 `/boot/uEnv.txt` 或 `/boot/extlinux/extlinux.conf`，添加 Overlay：
    ```bash
    overlays=rk3568-spi3-m1-cs0-spidev
    ```
    *重启后检查:* `ls /dev/spidev3.0` 存在即成功。

### 3. 编译与运行
```bash
# 1. 克隆项目
git clone [https://github.com/yourname/project-name.git](https://github.com/yourname/project-name.git)

# 2. 创建构建目录
mkdir build && cd build

# 3. 编译 (确保已配置好 Qt 环境变量)
qt-cmake ..
make -j4

# 4. 赋予 SPI 权限并运行
sudo chmod 666 /dev/spidev3.0
./ELE_Stimulation_System```
- **`IBackend` (Interface)**: 定义了 `start()`, `stop()`, `updateParam()` 等纯虚函数，实现策略模式。
    
- **`RK3568Backend`**: 真实硬件实现。
    
    - 封装 Linux `spidev` 驱动接口 (`ioctl`)。
        
    - 实现 SPI 协议的打包、解包与 CRC 校验。
        
    - **高性能 IO**: 采用 Direct IO 模式减少内核拷贝。
        
- **`WinBackend`**: 模拟器实现。
    
    - 内置信号发生器算法 (`sin` + `random noise`)，模拟真实负载下的波形与阻抗抖动。
        
    - 允许在无硬件环境下开发 95% 的业务功能。
**Worker Thread (Backend)**:

- 驻留在一个独立的 `QThread` 中。
    
- 运行高频定时器 (50Hz - 100Hz) 或阻塞式 SPI 读取循环。
    
- 数据通过 `QueuedConnection` 跨线程安全地传递给 UI 层。