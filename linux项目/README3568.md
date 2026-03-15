# 基于 RK3568 的工业级嵌入式 Linux 终端——从驱动到 OTA 的全生命周期

  

> 自研内核驱动 + A/B 分区 OTA 升级 + Buildroot 系统定制

  

---

  

## 一、项目概述

  

### 1.1 产品定义

  

一台**具备完整产品生命周期的嵌入式 Linux 终端**：不仅能采集数据和显示界面，还支持通过网络远程升级固件，且任何升级失败都能自动回滚——**永不变砖**。

  

这不是一个纯技术演示，而是一个**具备工业级可靠性**的完整产品。

  

### 1.2 核心亮点

  

| 亮点 | 简历关键词 |

|------|-----------|

| A/B 分区 OTA 远程升级 + 自动回滚 | U-Boot 改造、分区管理、固件升级 |

| UART Line Discipline 内核模块 | TTY 子系统、内核态协议解析 |

| I2C IIO 传感器驱动 | I2C 总线、IIO 框架、设备树 |

| PWM + Watchdog 驱动 | Platform 驱动、看门狗子系统 |

| Buildroot 系统深度定制 | 交叉编译、内核裁剪、系统构建 |

| 跨平台可移植（RK3568 → i.MX6ULL） | 设备驱动模型、分层抽象 |

  

### 1.3 目标指标

  

| 指标 | 基线（当前） | 目标 |

|------|------------|------|

| 内核镜像 | 22 MB | ≤ 10 MB |

| 根文件系统 | 1.4 GB | ≤ 50 MB |

| OTA 升级耗时 | — | ≤ 60s（含下载+写入+重启） |

| 升级失败恢复 | — | 全自动回滚，≤ 3 次重试 |

  

---

  

## 二、系统架构

  

```

┌──────────────────────────────────────────────────────┐

│               RK3568 Linux 终端                       │

│                                                      │

│  ┌────────┐  ┌─────────────┐  ┌───────────────────┐ │

│  │ Qt QML │  │ OTA 守护进程 │  │  健康监控服务      │ │

│  │ 界面    │  │ (ota_daemon)│  │  (读取 Procfs)    │ │

│  └───┬────┘  └──────┬──────┘  └────────┬──────────┘ │

│      │              │                   │            │

│  ════╪══════ 内核态 ═╪═══════════════════╪═══════════ │

│      │              │                   │            │

│ ┌────┴─────┐  ┌─────┴──────┐  ┌────────┴─────────┐ │

│ │UART Line │  │ USB/Block  │  │ Watchdog 驱动     │ │

│ │Discipline│  │ 设备驱动    │  │ (手写)            │ │

│ │(手写)    │  │ (内核已有)  │  │                   │ │

│ ├──────────┤  └────────────┘  ├───────────────────┤ │

│ │I2C IIO   │                  │ Procfs 统计接口   │ │

│ │BME280    │                  │ (手写)            │ │

│ │(手写)    │                  └───────────────────┘ │

│ ├──────────┤                                        │

│ │PWM 背光  │                                        │

│ │(手写)    │                                        │

│ └──────────┘                                        │

│                                                      │

│  存储分区布局：                                        │

│  ┌──────┬────────┬────────┬────────┬────────┬─────┐ │

│  │UBoot │Kernel_A│Kernel_B│RootFS_A│RootFS_B│Data │ │

│  │      │ (活跃)  │ (备用)  │ (活跃)  │ (备用)  │     │ │

│  └──────┴────────┴────────┴────────┴────────┴─────┘ │

└──────────┬───────────────────────────┬───────────────┘

      UART 串口                 以太网（OTA 下载通道）

           │                          │

    ┌──────┴──────┐          ┌────────┴────────┐

    │ STM32 C8T6   │          │ Ubuntu 虚拟机    │

    │ 传感器节点    │          │ HTTP 固件服务器  │

    └─────────────┘          └─────────────────┘

```

  

---

  

## 三、核心模块设计

  

### 3.1 模块一：A/B 分区 OTA 升级系统（⭐ 最核心亮点）

  

这是整个项目的灵魂，体现**产品级思维**。

  

#### 分区方案

  

```

eMMC / SD 卡分区表：

  

偏移       分区         大小        说明

0x0000     U-Boot      4 MB       Bootloader（不可升级区）

0x0400     env         1 MB       U-Boot 环境变量（boot_count 等）

0x0500     Kernel_A    16 MB      活跃内核 + DTB

0x1500     Kernel_B    16 MB      备用内核 + DTB

0x2500     RootFS_A    64 MB      活跃根文件系统

0x6500     RootFS_B    64 MB      备用根文件系统

0xA500     UserData    剩余空间    用户数据（不受 OTA 影响）

```

  

#### OTA 升级流程

  

```

  ┌─────────────────────────────────────┐

  │  OTA 守护进程 (ota_daemon)           │

  │  每 60 秒轮询一次固件服务器          │

  └──────────────┬──────────────────────┘

                 │

                 ▼

        有新版本？──── 否 ──→ 继续轮询

                 │

                 是

                 ▼

        HTTP 下载 update.pkg 到 /tmp/

                 │

                 ▼

        SHA256 校验 ──── 失败 ──→ 删除，记录日志

                 │

                 成功

                 ▼

        dd 写入备用分区（B 分区）

                 │

                 ▼

        修改 U-Boot 环境变量：

          boot_slot=B

          boot_count=0

          upgrade_pending=1

                 │

                 ▼

        reboot（重启）

                 │

                 ▼

  ┌──────────────┴──────────────────────┐

  │  U-Boot 启动逻辑                     │

  │                                     │

  │  读取 boot_slot → 从 B 分区加载内核  │

  │  boot_count++                       │

  │                                     │

  │  if boot_count > 3:                 │

  │      boot_slot=A  ← 自动回滚！       │

  │      upgrade_pending=0              │

  └──────────────┬──────────────────────┘

                 │

                 ▼ 成功启动

        用户态确认服务运行正常

        清除 upgrade_pending

        → 升级完成！

```

  

#### U-Boot 启动脚本修改（boot.cmd）

  

```bash

# 读取环境变量

if test "${boot_slot}" = "B"; then

    setenv kernel_part 3        # Kernel_B 分区号

    setenv rootfs_part 5        # RootFS_B 分区号

else

    setenv kernel_part 2        # Kernel_A 分区号

    setenv rootfs_part 4        # RootFS_A 分区号

fi

  

# 启动计数 + 回滚检测

setexpr boot_count ${boot_count} + 1

saveenv

  

if test ${boot_count} -gt 3; then

    echo "Boot failed 3 times, rolling back!"

    setenv boot_slot A

    setenv boot_count 0

    setenv kernel_part 2

    setenv rootfs_part 4

    saveenv

fi

  

# 加载内核和设备树，启动

load mmc 0:${kernel_part} ${kernel_addr} Image

load mmc 0:${kernel_part} ${fdt_addr} rk3568.dtb

setenv bootargs root=/dev/mmcblk0p${rootfs_part} rootfstype=ext4 ...

booti ${kernel_addr} - ${fdt_addr}

```

  

#### OTA 守护进程（用户态 C 程序）

  

```c

// ota_daemon.c 核心逻辑伪代码

int main() {

    while (1) {

        // 1. 检查服务器有没有新版本

        if (check_update("http://server:8080/version.json")) {

            // 2. 下载固件

            download("http://server:8080/update.pkg", "/tmp/update.pkg");

            // 3. SHA256 校验

            if (!verify_sha256("/tmp/update.pkg", expected_hash))

                continue;

            // 4. 写入备用分区

            write_to_partition("/tmp/update.pkg", get_standby_partition());

            // 5. 切换 U-Boot 启动分区

            set_uboot_env("boot_slot", get_standby_slot());

            set_uboot_env("boot_count", "0");

            set_uboot_env("upgrade_pending", "1");

            // 6. 重启

            reboot();

        }

        sleep(60);

    }

}

```

  

---

  

### 3.2 模块二：UART Line Discipline 驱动

  

（与之前方案相同，此处从简）

  

- **帧协议**：帧头(0xAA 0x55) + 类型(1B) + 长度(1B) + 数据(NB) + CRC16(2B)

- **内核模块**：注册自定义 line discipline，在 `receive_buf` 中用状态机解析帧

- **对接 STM32**：C8T6 每 100ms 上报模拟传感器数据

- **Procfs**：`/proc/uart_sensor_stats` 显示收帧计数、CRC 错误计数

  

---

  

### 3.3 模块三：I2C IIO 驱动（BME280）

  

- 手写 I2C client 驱动，注册到 IIO 子系统

- 设备树绑定 `compatible = "my,bme280-custom"`

- 通过 `/sys/bus/iio/devices/` 读取温湿度气压

  

---

  

### 3.4 模块四：PWM 背光 + Watchdog 驱动

  

- **PWM**：Platform 驱动，sysfs 接口控制屏幕背光

- **Watchdog**：手写 watchdog 驱动，注册到 watchdog 子系统

  - 用户态必须定期"喂狗"（写入 `/dev/watchdog`）

  - 如果 OTA 升级后系统死机 → 无人喂狗 → 硬件自动复位 → U-Boot 检测 boot_count 超限 → 回滚

  - 这形成了 OTA 安全网的**最后一道防线**

  

---

  

### 3.5 模块五：Qt QML 界面（辅助展示）

  

极简界面，展示：

- 传感器实时数据（温度/湿度/气压）

- 系统状态（当前活跃分区 A/B、固件版本号）

- OTA 状态（检查中/下载中/升级成功/已回滚）

- 背光调节滑块

  

---

  

## 四、开发时间规划

  

| 阶段 | 时间 | 内容 | 产出 |

|------|------|------|------|

| **系统构建** | 3月上（2周） | 内核/rootfs 裁剪 + 自定义 init | 精简系统 |

| **OTA 系统** | 3月底-4月中（3周） | 分区设计 + U-Boot 改造 + OTA 守护进程 | 可升级可回滚 |

| **UART 驱动** | 4月中-4月底（2周） | Line Discipline + STM32 联调 | 通信链路 |

| **I2C/PWM/WDG** | 5月上（2周） | BME280 + 背光 + 看门狗 | 三个驱动 |

| **Qt + 集成** | 5月中-5月底（2周） | 界面 + 系统集成 | 完整产品 |

| **收尾** | 6月上（2周） | GitHub + 文档 + 视频 + 简历 | 展示物 |

  

---

  

## 五、硬件清单

  

| 物品 | 价格 | 状态 |

|------|------|------|

| RK3568 开发板（正点原子） | — | ✅ 已有 |

| MIPI / HDMI 屏幕 | — | ✅ 已有 |

| STM32 C8T6 | — | ✅ 已有 |

| BME280 传感器 | ¥8 | ❌ 需购买 |

| 网线 | ¥5 | 可能有 |

| **总计** | **≈ ¥15** | |

  

---

  

## 六、项目仓库结构

  

```

rk3568-linux-terminal/

├── README.md

├── buildroot/

│   ├── configs/rk3568_minimal_defconfig

│   └── overlay/etc/init.d/rcS

├── kernel/

│   └── configs/rk3568_minimal_defconfig

├── drivers/

│   ├── uart_sensor_ldisc/        # UART Line Discipline

│   ├── my_bme280/                # I2C IIO 驱动

│   ├── my_pwm_backlight/         # PWM 背光驱动

│   └── my_watchdog/              # Watchdog 驱动

├── dts/

│   └── rk3568-custom.dtsi        # 设备树片段

├── ota/

│   ├── ota_daemon.c              # OTA 守护进程

│   ├── ota_config.h              # 配置（服务器地址、分区路径）

│   ├── uboot_env.c               # U-Boot 环境变量读写

│   └── sha256.c                  # 固件校验

├── uboot/

│   └── boot.cmd                  # 修改后的 U-Boot 启动脚本

├── stm32_firmware/               # STM32 C8T6 从机固件

├── qt_app/                       # Qt QML 界面

│   ├── main.cpp

│   ├── main.qml

│   └── CMakeLists.txt

└── docs/

    ├── kernel_trim_analysis.md   # 内核裁剪记录

    ├── ota_design.md             # OTA 系统设计文档

    └── build_guide.md            # 编译指南

```

  

---

  

## 七、简历项目描述

  

> **基于 RK3568 的工业级嵌入式 Linux 终端（含 A/B 分区 OTA）** ｜ 2026.03 - 2026.06

> - 使用 Buildroot 从零定制嵌入式 Linux 系统，深度裁剪内核与根文件系统

> - **设计并实现 A/B 分区 OTA 固件升级系统**：改造 U-Boot 启动逻辑，实现网络远程升级、固件原子切换与 boot_count 自动回滚机制，确保设备在升级失败时永不变砖

> - 编写 **UART Line Discipline 内核模块**，在内核态实现自定义协议的帧同步、CRC16 校验与状态机解析

> - 手写 BME280 **I2C IIO 驱动**与 **PWM 背光驱动**，覆盖 I2C / PWM 子系统与设备树编写

> - 编写 **Watchdog 内核驱动**，与 OTA 回滚机制联动，构成系统级死机恢复安全网

> - 通过 Procfs 接口暴露驱动运行时统计，实现系统健康可观测性

  

---

  

## 八、验证计划

  

| 场景 | 验证方法 | 预期结果 |

|------|---------|---------|

| 正常 OTA 升级 | 服务器放新固件，板子自动下载升级重启 | 从 A 切到 B，版本号更新 |

| 固件损坏回滚 | 故意上传坏固件，板子升级后无法启动 | 3 次重启后自动回滚到 A |

| 看门狗恢复 | 升级后进程死锁，无人喂狗 | 硬件复位 → 回滚 |

| 断电保护 | 写入分区过程中拔电 | 旧分区不受影响，系统正常启动 |

| 驱动功能 | STM32 发送数据帧 | Linux 端正确解析并显示 |

| 传感器读取 | 读取 IIO sysfs 节点 | 温湿度数据合理 |