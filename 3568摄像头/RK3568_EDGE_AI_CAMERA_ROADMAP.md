# 基于 RK3568 的边缘 AI 网络摄像机开发路线

## 1. 项目概述

本项目基于正点原子 ATOMPI RK3568 开发板、OV13850 MIPI 摄像头和 10 寸 MIPI 显示屏，设计并实现一个具备本地视频预览、网络推流、边缘 AI 检测、事件录像和远程管理能力的嵌入式 Linux 音视频系统。

项目初始形态是一个“摄像头采集 + MIPI 屏显示 + 网络推流”的嵌入式音视频终端；后续逐步扩展为具备目标检测、事件告警、录像管理、Web 管理后台、双向音视频通信能力的边缘 AI 网络摄像机。

项目最终目标不是单纯完成一个摄像头 demo，而是形成一个接近真实产品形态的嵌入式音视频系统。

---

## 2. 硬件平台

### 2.1 已有硬件

| 硬件 | 用途 |
|---|---|
| 正点原子 ATOMPI RK3568 开发板 | 主控平台，运行嵌入式 Linux |
| OV13850 MIPI 摄像头 | 视频采集输入 |
| 10 寸 MIPI 显示屏 | 本地视频预览和 UI 显示 |

### 2.2 后续建议增加的硬件

| 硬件 | 用途 |
|---|---|
| USB 麦克风或板载麦克风模块 | 音频采集 |
| USB 声卡 / 喇叭 / 耳机 | 音频播放 |
| 网线或 Wi-Fi 模块 | 网络推流和远程访问 |
| 按键模块 | 本地控制，例如拍照、录像、切换模式 |
| 蜂鸣器或 LED | 本地告警提示 |
| TF 卡或 eMMC 存储 | 保存录像、截图、日志和模型文件 |

---

## 3. 项目定位

项目名称建议：

> 基于 RK3568 的边缘 AI 网络摄像机

或：

> 基于 RK3568 的边缘智能视频采集与推流系统

项目核心功能包括：

1. OV13850 MIPI 摄像头视频采集
2. 10 寸 MIPI 屏本地实时显示
3. H.264/H.265 硬件编码
4. RTSP / WebRTC / UDP 网络推流
5. RKNN / OpenCV 边缘 AI 检测
6. 检测框和状态信息叠加显示
7. 事件触发截图和录像
8. Web 管理后台
9. 后续扩展双向音视频通信

---

## 4. 项目整体架构

```text
OV13850 MIPI 摄像头
        │
        ▼
Linux Kernel Camera Driver / Device Tree
        │
        ▼
V4L2 / Media Controller / ISP
        │
        ├──────────────► 本地预览链路
        │                    │
        │                    ▼
        │              RGA / DRM / Wayland / Qt
        │                    │
        │                    ▼
        │              10 寸 MIPI 屏显示
        │
        ├──────────────► 编码推流链路
        │                    │
        │                    ▼
        │              MPP H.264/H.265 编码
        │                    │
        │                    ▼
        │              RTSP / WebRTC / UDP / HTTP-FLV
        │                    │
        │                    ▼
        │              主机 / 浏览器 / VLC 观看
        │
        └──────────────► 边缘 AI 链路
                             │
                             ▼
                       RGA 缩放 / 格式转换
                             │
                             ▼
                       RKNN / OpenCV 推理
                             │
                             ▼
                       检测结果后处理
                             │
                             ├──► 检测框叠加显示
                             ├──► 事件录像
                             ├──► 截图保存
                             └──► MQTT / HTTP / WebSocket 告警
```

后续加入双向音视频后，架构可扩展为：

```text
本地摄像头 → 编码 → 网络 → 主机
主机视频 → 网络 → 解码 → MIPI 屏
本地音频 → 编码 → 网络 → 主机
主机音频 → 网络 → 解码 → 扬声器
```

---

## 5. 技术栈规划

### 5.1 底层驱动与系统

| 模块 | 技术点 |
|---|---|
| 摄像头驱动 | OV13850 sensor driver、V4L2 subdev、media controller |
| MIPI CSI | CSI-2、D-PHY、ISP pipeline、device tree endpoint |
| MIPI DSI 屏 | DRM panel、backlight、touch 可选、display timing |
| 设备树 | camera、mipi-csi、mipi-dsi、i2c、clock、gpio、regulator |
| 内核调试 | dmesg、debugfs、ftrace、dynamic debug、media-ctl |
| 模块开发 | Kbuild、LKM、sysfs、procfs、char device，可作为学习扩展 |

### 5.2 音视频应用层

| 模块 | 技术点 |
|---|---|
| 视频采集 | V4L2、GStreamer v4l2src |
| 图像处理 | RGA、OpenCV、格式转换、缩放、旋转 |
| 视频编码 | Rockchip MPP、H.264、H.265 |
| 视频解码 | Rockchip MPP decoder |
| 本地显示 | DRM/KMS、kmssink、Wayland、Qt、SDL |
| 网络推流 | RTSP、RTP/UDP、WebRTC、HTTP-FLV 可选 |
| 音频 | ALSA、Opus、AAC、PCM、AEC 可选 |

### 5.3 边缘 AI

| 模块 | 技术点 |
|---|---|
| 模型部署 | RKNN Toolkit2、ONNX 转 RKNN |
| 推理运行 | RKNN Runtime、NPU 调用 |
| 模型选择 | YOLOv5n、YOLOv8n、MobileNet、SCRFD、RetinaFace |
| 后处理 | NMS、bbox 转换、置信度过滤 |
| 显示叠加 | OpenCV 绘制、RGA 合成、GStreamer overlay |
| 事件逻辑 | 人体检测、区域入侵、越线检测、事件录像 |

---

## 6. 开发总路线

项目建议按照“先打通硬件，再打通音视频链路，再加入 AI，再产品化”的顺序推进。

```text
阶段 0：开发环境准备
阶段 1：MIPI 屏点亮
阶段 2：OV13850 摄像头出图
阶段 3：本地实时预览
阶段 4：视频编码与主机端播放
阶段 5：RTSP / WebRTC 网络摄像机
阶段 6：边缘 AI 目标检测
阶段 7：检测框叠加与事件录像
阶段 8：Web 管理后台
阶段 9：双向音视频扩展
阶段 10：系统优化与产品化收尾
```

---

## 7. 阶段 0：开发环境准备

### 7.1 目标

搭建可重复开发、编译、烧录、调试的嵌入式 Linux 开发环境。

### 7.2 主要任务

1. 获取 ATOMPI RK3568 官方 SDK 或 Buildroot / Debian 镜像
2. 搭建交叉编译工具链
3. 熟悉镜像烧录流程
4. 配置串口调试终端
5. 配置 SSH 登录
6. 准备主机端调试工具
7. 建立项目目录结构

### 7.3 主机端建议工具

```text
adb / ssh / scp
minicom / picocom / mobaxterm
v4l-utils
media-ctl
ffmpeg / ffplay
VLC
GStreamer
Wireshark
Git
VS Code
```

### 7.4 板端建议工具

```text
v4l2-ctl
media-ctl
gst-launch-1.0
gst-inspect-1.0
ffmpeg / ffplay 可选
rknn_runtime
rga demo
mpp demo
dmesg
trace-cmd 可选
```

### 7.5 阶段产出

- 可以通过串口进入系统
- 可以通过 SSH 访问开发板
- 可以交叉编译并运行 hello world
- 可以查看内核日志和设备节点

---

## 8. 阶段 1：MIPI 屏点亮

### 8.1 目标

让 10 寸 MIPI 屏幕在 Linux 系统中正常显示。

### 8.2 主要任务

1. 确认屏幕型号和分辨率
2. 确认 MIPI DSI lane 数量
3. 确认供电、背光、reset GPIO
4. 修改或确认设备树中的 panel 节点
5. 确认 DRM 设备节点生成
6. 测试 framebuffer / DRM / Wayland 显示

### 8.3 调试重点

```text
/dev/dri/card0
/dev/fb0
dmesg | grep -i drm
dmesg | grep -i dsi
dmesg | grep -i panel
```

重点关注：

- panel 是否 probe 成功
- 背光是否正常
- 分辨率是否正确
- 是否存在花屏、偏色、闪屏
- 是否存在显示方向问题

### 8.4 阶段产出

- MIPI 屏可以显示系统桌面或测试图像
- 可以运行简单图形程序
- 可以用 DRM/KMS 或 Wayland 输出画面

---

## 9. 阶段 2：OV13850 摄像头出图

### 9.1 目标

让 OV13850 MIPI 摄像头在 Linux 下被识别，并能通过 V4L2 采集图像。

### 9.2 主要任务

1. 确认摄像头模组接口、电源、reset、pwdn、mclk
2. 确认 OV13850 I2C 地址
3. 检查 kernel 中是否已有 OV13850 driver
4. 配置 sensor 节点、mipi csi 节点、isp 节点
5. 使用 media-ctl 查看 media graph
6. 使用 v4l2-ctl 测试采集

### 9.3 关键命令

```bash
media-ctl -p
v4l2-ctl --list-devices
v4l2-ctl -d /dev/video0 --list-formats-ext
v4l2-ctl -d /dev/video0 --all
v4l2-ctl -d /dev/video0 --stream-mmap --stream-count=100 --stream-to=test.raw
```

### 9.4 调试重点

```text
dmesg | grep -i ov13850
dmesg | grep -i csi
dmesg | grep -i mipi
dmesg | grep -i isp
dmesg | grep -i v4l2
```

重点关注：

- I2C 是否能读到 sensor ID
- clock 是否配置正确
- reset / pwdn GPIO 是否正确
- MIPI lane 数量是否匹配
- endpoint remote-endpoint 是否连接正确
- sensor 输出格式是否被 ISP 支持

### 9.5 阶段产出

- `/dev/videoX` 设备正常生成
- `media-ctl -p` 能看到完整 pipeline
- 可以抓取 raw/yuv 图像
- 摄像头稳定出图

---

## 10. 阶段 3：本地实时预览

### 10.1 目标

实现摄像头画面实时显示到 10 寸 MIPI 屏幕。

### 10.2 推荐路线

先用 GStreamer 快速验证：

```text
v4l2src → videoconvert/rga → kmssink/waylandsink
```

如果性能不够，再切换到更底层方案：

```text
V4L2 → RGA → DRM/KMS
```

### 10.3 示例 pipeline 方向

实际命令需要根据板端插件和格式调整：

```bash
gst-launch-1.0 v4l2src device=/dev/video0 ! videoconvert ! waylandsink
```

或：

```bash
gst-launch-1.0 v4l2src device=/dev/video0 ! videoconvert ! kmssink
```

### 10.4 需要记录的指标

| 指标 | 目标 |
|---|---|
| 分辨率 | 先 720p，后续 1080p |
| 帧率 | 25 fps 或 30 fps |
| 预览延迟 | 尽量低于 150 ms |
| CPU 占用 | 尽量避免纯 CPU 色彩转换 |
| 稳定性 | 连续运行 1 小时无异常 |

### 10.5 阶段产出

- 摄像头画面能在 MIPI 屏实时显示
- 可以显示 FPS、分辨率、格式等状态信息
- 明确当前 pipeline 的 CPU 占用和延迟

---

## 11. 阶段 4：视频编码与主机端播放

### 11.1 目标

将摄像头采集到的视频使用 RK3568 硬件编码为 H.264/H.265，并发送到主机播放。

### 11.2 基础链路

```text
OV13850 → V4L2 → MPP Encoder → RTP/UDP/文件 → 主机播放
```

### 11.3 开发任务

1. 验证 Rockchip MPP 编码 demo
2. 验证 GStreamer 中 Rockchip 编码插件
3. 完成 H.264 文件保存
4. 完成 UDP/RTP 发送
5. 主机使用 VLC / ffplay 播放
6. 调整码率、GOP、帧率、分辨率

### 11.4 主机端播放方式

```bash
ffplay udp://0.0.0.0:5000
```

或者使用 VLC 打开对应 UDP / RTSP 地址。

### 11.5 需要关注的问题

- 编码器是否真正走硬件 MPP
- 输入格式是否需要 RGA 转换
- 是否存在丢帧
- 码率是否稳定
- I 帧间隔是否合理
- 网络传输是否稳定

### 11.6 阶段产出

- 可以将摄像头画面编码为 H.264/H.265
- 主机可以实时播放开发板视频
- 可以调节码率、帧率、分辨率

---

## 12. 阶段 5：RTSP / WebRTC 网络摄像机

### 12.1 目标

把开发板变成一个可以被主机、浏览器或 VLC 访问的网络摄像机。

### 12.2 RTSP 路线

RTSP 优点：

- 实现简单
- VLC 支持好
- 适合摄像头项目
- 调试方便

推荐先实现 RTSP：

```text
V4L2 → MPP H.264 Encoder → RTSP Server → VLC/主机
```

### 12.3 WebRTC 路线

WebRTC 优点：

- 低延迟
- 浏览器原生支持
- 适合可视对讲和远程交互

但难点更多：

- signaling
- NAT 穿透
- jitter buffer
- 音视频同步
- 编码参数适配

建议在 RTSP 稳定后再加入 WebRTC。

### 12.4 阶段产出

- VLC 可以通过 RTSP 地址查看摄像头画面
- 浏览器可以通过 WebRTC 或 Web 页面查看画面
- 网络中断后系统可以恢复

---

## 13. 阶段 6：边缘 AI 目标检测

### 13.1 目标

在 RK3568 本地完成 AI 推理，不依赖主机或云端判断视频内容。

### 13.2 推荐模型

| 模型 | 用途 |
|---|---|
| YOLOv5n / YOLOv8n | 通用目标检测 |
| MobileNet-SSD | 轻量目标检测 |
| SCRFD / RetinaFace | 人脸检测 |
| 自定义安全帽模型 | 工业安全监控 |

### 13.3 AI 处理链路

```text
摄像头帧
  ↓
RGA 缩放到模型输入尺寸
  ↓
颜色格式转换
  ↓
RKNN Runtime 推理
  ↓
后处理 NMS
  ↓
输出检测框和类别
```

### 13.4 开发任务

1. 在 PC 上准备 ONNX 模型
2. 使用 RKNN Toolkit2 转换模型
3. 在板端部署 RKNN Runtime
4. 使用测试图片验证推理
5. 接入实时摄像头帧
6. 输出检测框、类别、置信度
7. 统计推理 FPS

### 13.5 需要记录的性能指标

| 指标 | 说明 |
|---|---|
| 模型输入尺寸 | 例如 320x320、640x640 |
| 推理耗时 | 单帧推理 ms |
| AI FPS | 每秒推理次数 |
| 视频 FPS | 实际显示帧率 |
| CPU 占用 | 后处理和数据搬运开销 |
| NPU 占用 | 可通过相关工具观察 |
| 内存占用 | 模型和 buffer 占用 |

### 13.6 阶段产出

- RK3568 本地可以完成目标检测
- 能输出检测结果 JSON
- 可以在日志中看到类别、置信度、坐标

---

## 14. 阶段 7：检测框叠加与事件录像

### 14.1 目标

让 AI 检测结果真正体现在产品功能中，而不只是打印日志。

### 14.2 显示叠加

MIPI 屏显示：

```text
实时画面 + 检测框 + 类别 + 置信度 + FPS + 网络状态
```

可选实现方式：

1. OpenCV 绘制后送显示
2. GStreamer overlay
3. RGA 合成
4. DRM plane overlay
5. Qt UI 叠加

初期推荐 OpenCV 或 GStreamer，后期优化到 RGA/DRM。

### 14.3 事件录像逻辑

事件示例：

- 检测到人
- 检测到车辆
- 检测到人进入禁区
- 检测到未戴安全帽
- 检测到人脸
- 检测到运动目标

事件触发后：

```text
保存截图
保存事件视频
写入事件数据库
通过 MQTT/HTTP/WebSocket 通知主机
MIPI 屏显示告警状态
```

### 14.4 推荐事件数据格式

```json
{
  "timestamp": "2026-05-26 20:30:00",
  "event_type": "person_detected",
  "confidence": 0.92,
  "bbox": [120, 80, 300, 420],
  "snapshot": "events/20260526_203000.jpg",
  "video": "events/20260526_203000.mp4"
}
```

### 14.5 阶段产出

- MIPI 屏可以显示检测框
- 检测到目标后可以自动截图
- 检测到事件后可以保存录像片段
- 事件信息可以被主机读取

---

## 15. 阶段 8：Web 管理后台

### 15.1 目标

让项目从板端 demo 变成可管理设备。

### 15.2 Web 后台功能

1. 实时视频预览
2. AI 检测开关
3. 模型选择
4. 录像文件列表
5. 事件记录查看
6. 截图查看和下载
7. 摄像头参数配置
8. 码率、帧率、分辨率配置
9. 系统状态查看
10. 日志查看

### 15.3 推荐技术栈

轻量方案：

```text
后端：Python Flask / FastAPI
前端：HTML + JavaScript
视频：MJPEG / HLS / WebRTC
事件推送：WebSocket
```

更产品化方案：

```text
后端：Go / C++
前端：Vue / React
视频：WebRTC / RTSP 转 Web
事件：MQTT / WebSocket
数据库：SQLite
```

### 15.4 阶段产出

- 主机浏览器可以访问开发板 Web 页面
- 可以查看实时视频和事件记录
- 可以配置基础参数

---

## 16. 阶段 9：双向音视频扩展

### 16.1 目标

在边缘 AI 网络摄像机基础上，扩展成可视对讲或视频会议终端。

### 16.2 视频扩展

当前已有：

```text
RK3568 摄像头 → 主机
```

新增：

```text
主机视频 → RK3568 → MIPI 屏显示
```

显示模式：

```text
远端视频全屏 + 本地摄像头小窗
```

### 16.3 音频扩展

新增链路：

```text
RK3568 麦克风 → 主机播放
主机麦克风 → RK3568 喇叭播放
```

### 16.4 技术路线

建议顺序：

```text
UDP/RTP 单向视频接收
RTSP 拉流显示
双画面合成
ALSA 音频采集播放
WebRTC 双向音视频
AEC 回声消除
```

### 16.5 阶段产出

- MIPI 屏可以显示主机传来的视频
- 可以同时显示远端大画面和本地小窗
- 可以完成基础双向音频
- 后续可升级为 WebRTC 可视对讲

---

## 17. 阶段 10：系统优化与产品化

### 17.1 性能优化

重点优化：

1. 减少 CPU 色彩转换
2. 尽量使用 RGA 做缩放和格式转换
3. 尽量使用 MPP 硬件编码/解码
4. 使用 DMA-BUF 减少内存拷贝
5. 避免 AI 推理阻塞显示和编码
6. 视频采集、AI、编码、显示分线程处理
7. 控制 buffer 队列长度，降低延迟

### 17.2 稳定性优化

需要考虑：

- 摄像头掉线恢复
- 网络断开重连
- 推流异常恢复
- 录像文件异常保护
- 看门狗
- 日志轮转
- 磁盘空间不足处理
- 进程崩溃自动拉起

### 17.3 系统服务化

将核心程序做成 systemd 服务：

```text
edge-ai-camera.service
```

支持：

```bash
systemctl start edge-ai-camera
systemctl stop edge-ai-camera
systemctl restart edge-ai-camera
systemctl status edge-ai-camera
```

### 17.4 OTA 与配置管理

后续可加入：

- 配置文件管理
- 模型文件管理
- Web 上传模型
- OTA 升级
- 日志导出
- 恢复出厂设置

---

## 18. 建议的软件模块划分

建议项目应用层按模块拆分：

```text
edge-ai-camera/
├── camera/          # V4L2 摄像头采集
├── display/         # DRM/Wayland/Qt 显示
├── encoder/         # MPP 编码
├── decoder/         # MPP 解码，后续双向视频使用
├── streamer/        # RTSP/WebRTC/UDP 推流
├── ai/              # RKNN/OpenCV 推理
├── recorder/        # 录像和截图
├── event/           # 事件管理和告警
├── web/             # Web 管理后台
├── audio/           # ALSA 音频采集播放
├── config/          # 配置文件
├── system/          # systemd、日志、看门狗
└── docs/            # 文档
```

如果使用 C/C++，可进一步规划为：

```text
src/
├── main.cpp
├── camera_v4l2.cpp
├── display_drm.cpp
├── mpp_encoder.cpp
├── rga_processor.cpp
├── rknn_detector.cpp
├── rtsp_server.cpp
├── recorder.cpp
├── event_manager.cpp
└── web_server.cpp
```

---

## 19. 推荐的里程碑计划

### Milestone 1：硬件基础验证

目标：屏幕亮、摄像头出图。

交付物：

- MIPI 屏显示正常
- OV13850 被系统识别
- 能抓取摄像头图像
- 记录 DTS 修改点和调试日志

### Milestone 2：本地预览 demo

目标：摄像头画面实时显示到 MIPI 屏。

交付物：

- 本地预览程序或 GStreamer pipeline
- FPS 统计
- CPU 占用记录

### Milestone 3：网络摄像机 demo

目标：主机可以实时观看开发板摄像头画面。

交付物：

- H.264/H.265 编码
- RTSP 或 UDP 推流
- VLC/ffplay 播放验证

### Milestone 4：边缘 AI demo

目标：开发板本地完成目标检测。

交付物：

- RKNN 模型部署
- 摄像头实时检测
- 检测结果日志输出

### Milestone 5：AI 叠加显示和事件截图

目标：让 AI 结果体现在屏幕和事件系统中。

交付物：

- MIPI 屏检测框叠加
- 目标检测截图
- JSON 事件记录

### Milestone 6：事件录像和 Web 后台

目标：具备基本产品形态。

交付物：

- 事件录像
- Web 页面查看实时视频
- Web 页面查看事件记录

### Milestone 7：双向音视频扩展

目标：接近可视对讲终端。

交付物：

- 主机视频显示到 MIPI 屏
- 本地小窗 + 远端大窗
- 基础音频采集和播放

---

## 20. 典型应用场景

### 20.1 边缘 AI 网络摄像机

功能：

- 本地摄像头采集
- MIPI 屏实时预览
- AI 检测人/车/物体
- RTSP 推流
- 事件录像
- Web 后台管理

适合作为项目主线。

### 20.2 智能门禁 / 可视对讲

功能：

- 人脸检测
- 访客抓拍
- 主机视频下发显示
- 双向音频
- 本地屏幕 UI

适合作为后续增强方向。

### 20.3 工业安全监控

功能：

- 安全帽检测
- 区域入侵检测
- 烟火检测
- MQTT 告警
- 事件录像

适合突出边缘计算价值。

### 20.4 智能看护终端

功能：

- 人体检测
- 跌倒检测
- 异常静止检测
- 本地告警
- 远程通知

适合民用看护场景。

---

## 21. 调试与验证方法

### 21.1 摄像头链路验证

```bash
media-ctl -p
v4l2-ctl --list-devices
v4l2-ctl -d /dev/video0 --list-formats-ext
v4l2-ctl -d /dev/video0 --stream-mmap --stream-count=100
```

### 21.2 显示链路验证

```bash
ls /dev/dri/
dmesg | grep -i drm
dmesg | grep -i panel
dmesg | grep -i dsi
```

### 21.3 编码链路验证

```bash
gst-inspect-1.0 | grep -i mpp
gst-inspect-1.0 | grep -i h264
gst-inspect-1.0 | grep -i rga
```

### 21.4 内核与驱动调试

```bash
dmesg -w
cat /proc/interrupts
cat /sys/kernel/debug/clk/clk_summary
cat /sys/kernel/debug/gpio
```

需要深入定位时，可以使用：

```bash
mount -t debugfs none /sys/kernel/debug
echo function > /sys/kernel/debug/tracing/current_tracer
cat /sys/kernel/debug/tracing/trace
```

对于自写或修改的内核模块，需要遵循 Kbuild 方式编译，并使用 `modinfo`、`insmod`、`rmmod`、`lsmod`、`dmesg` 验证。

### 21.5 性能验证

需要长期记录：

```text
CPU 占用
内存占用
视频帧率
编码延迟
推流延迟
AI 推理耗时
NPU 使用情况
温度
连续运行稳定性
```

---

## 22. 关键风险与应对策略

### 22.1 摄像头无法出图

可能原因：

- I2C 地址错误
- reset/pwdn GPIO 配置错误
- mclk 频率错误
- MIPI lane 数量错误
- endpoint 连接错误
- sensor 输出格式不匹配

应对策略：

- 先确认 dmesg 是否读到 sensor ID
- 用示波器或逻辑分析仪确认关键引脚
- 用 media-ctl 检查 pipeline
- 逐步降低分辨率和帧率测试

### 22.2 MIPI 屏显示异常

可能原因：

- timing 错误
- 背光 GPIO 错误
- reset 时序错误
- DSI lane 配置错误
- panel init sequence 不匹配

应对策略：

- 先确认 panel probe
- 检查 DRM 日志
- 使用厂商提供的 dts 和驱动作为参考

### 22.3 视频延迟过高

可能原因：

- buffer 队列过长
- CPU 参与过多格式转换
- 没有使用硬件编码
- 网络缓存过大
- 显示同步策略导致延迟

应对策略：

- 减少 pipeline queue
- 使用 RGA/MPP
- 降低编码 B 帧或关闭 B 帧
- 调整 GOP 和码率
- 使用低延迟协议

### 22.4 AI 推理影响视频流畅度

可能原因：

- 每帧都推理导致负载过高
- 图像预处理走 CPU
- 后处理耗时过大
- 线程之间阻塞

应对策略：

- AI 每隔 N 帧推理一次
- 显示和编码走独立线程
- RGA 预处理
- 使用轻量模型
- 控制输入尺寸

---

## 23. 推荐学习顺序

建议按以下顺序补充知识：

1. 嵌入式 Linux 基础
2. 设备树基础
3. V4L2 和 media controller
4. DRM/KMS 显示基础
5. GStreamer 基础
6. Rockchip MPP / RGA
7. RTSP / RTP / WebRTC 基础
8. OpenCV 图像处理
9. RKNN 模型部署
10. ALSA 音频开发
11. systemd 服务和日志管理
12. Linux 驱动调试、ftrace、debugfs

---

## 24. 当前建议的第一步

当前最优先完成的是硬件链路验证：

```text
1. 点亮 10 寸 MIPI 屏
2. 让 OV13850 摄像头被内核识别
3. 使用 v4l2-ctl 抓取摄像头帧
4. 使用 GStreamer 将摄像头画面显示到 MIPI 屏
```

只有摄像头和屏幕都稳定后，再进入编码、推流和 AI 阶段。

---

## 25. 最终项目目标总结

最终系统应具备以下能力：

```text
摄像头采集：OV13850 MIPI 摄像头稳定采集
本地显示：10 寸 MIPI 屏实时预览
网络推流：主机可以实时查看视频流
硬件编码：使用 RK3568 MPP 完成 H.264/H.265 编码
边缘 AI：使用 RKNN 在板端完成目标检测
事件处理：检测到目标后截图、录像、记录事件
远程管理：主机浏览器可以查看状态和配置参数
可扩展通信：后续支持主机视频下发和双向音频
系统稳定：支持开机自启、异常恢复、日志记录
```

这个项目可以从一个基础摄像头预览 demo 逐步演进为一个完整的嵌入式边缘 AI 音视频终端，既能覆盖嵌入式 Linux 驱动调试，也能覆盖应用层音视频开发、网络传输、AI 推理和产品化系统集成。
