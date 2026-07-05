# 1. 概述

!!! abstract "本手册范围"
    覆盖 **从数采夹爪硬件 → 数据落盘为 `LeRobotDataset`** 的完整链路:
    概述/硬件 → 环境安装 → 软件使用采集 → 数据落盘与介绍。
    **模型训练、推理、部署不在范围内。**

## 1.1 TacCap-Gripper 是什么

**TacCap-Gripper**(TacCap = *Tactile Capture* Gripper,**对外市场名:千觉 XTac UMI G1**)
是 XenseRobotics 的**手持式 UMI 主夹爪**,专为多模态触觉数据采集设计。产品定位为
**面向机器人操作学习的穿戴式视触觉多模态数据采集夹爪**,更多产品价值见 [产品亮点](highlights.md)。
单个夹爪单元集成:

| 部件 | 说明 | 采样率 |
|---|---|---|
| 电机夹爪(motor jaw) | 采集时**不上电**,由操作员手动带动 | — |
| 编码器(Encoder) | 夹爪开合角度,标定后闭合=0、最大 ~1.7 rad(~97°) | 100 Hz |
| IMU | 加速度 / 角速度 / 磁力 / 温度 | 100 Hz |
| 双视触觉传感器(GSPS,左右指各一) | 视触觉图像,校正后约 `(400, 700, 3)` | ~30 Hz |
| 腕部相机(XC,UVC) | 手腕视角 RGB | ~30 Hz |

!!! warning "本设备是被动 / 自驱动的"
    采集时 `send_action()` 是**空操作**,电机永不使能。操作员**手持夹爪机械地**
    带动夹爪走完演示动作——因此**没有独立遥操作端**,`lerobot-record` 允许
    `teleop=None`,命令行上**不需要任何 `--teleop.*` 参数**。

## 1.2 数采系统组成

一次完整采集涉及 4 个部分协同:

```mermaid
flowchart TB
    subgraph 硬件
      G[TacCap-Gripper<br/>夹爪+触觉+腕相机+IMU]
      P[Pico4 Ultra<br/>独立运动追踪器]
    end
    subgraph 主机
      PS[XenseVR PC Service<br/>守护进程]
      SDK[xense.taccap SDK<br/>xensesdk 成像]
      LR[lerobot-record<br/>taccap_gripper 机器人类]
    end
    G -- USB / 串口+UVC --> SDK
    P -- 无线 --> PS
    PS -- 位姿 --> LR
    SDK -- 观测 --> LR
    LR --> DS[(LeRobotDataset<br/>parquet + mp4)]
```

- **夹爪** 通过 `xense.taccap` SDK 读取(串口 `/dev/ttyACM*` + UVC `/dev/video*`)。
- **Pico4 Ultra 运动追踪器**装在夹爪顶部,经 `xensevr_pc_service_sdk` 由
  `Pico4TrackerReader` 读取 6-DoF 位姿。
- **XenseVR PC Service** 是位姿数据的主机守护进程,追踪器与它通信。
- **lerobot-record** 把观测(t-1 帧)与动作(t 帧位姿 + 归一化夹爪开度)配对,写出数据集。

## 1.3 系统架构与数据流

`xense.taccap` 是纯**设备访问层**,不做数据集录制;录制、时间对齐、分集等由
`xense-taccap-lerobot` 上层完成。分层如下:

```mermaid
flowchart TB
    U[数采脚本 / lerobot-record] --> AGG[LeaderGripper 聚合对象<br/>start/stop_streaming]
    AGG --> C1[IMU]
    AGG --> C2[Encoder]
    AGG --> C3[Camera 腕相机]
    C1 & C2 --> T[异步 Transport<br/>ACK 匹配 · 后台读线程]
    T --> BUS[协议 + 串口总线<br/>CRC16 · 字节填充 · termios 3Mbps]
    BUS --> K[/dev/ttyACM · CH343 3Mbps/]
    C3 --> V[/dev/video · UVC/]
    K --> HW[TC-GU-01 夹爪硬件<br/>STM32H562]
```

!!! note "触觉成像在 Python 层"
    自 SDK 0.1.4 起,视触觉(OG)采集/校正**不在 C++ SDK 内**,而是在 Python 层
    通过 `xensesdk` wheel 完成;`xense.taccap` 只负责**夹爪协议 + 腕部相机**。

**每帧最终会记录**:Pico4 位姿(`tcp.*`)、归一化夹爪开度(`gripper.pos`)、
可选 IMU、左右触觉图、腕相机图——详见 [5.4 每帧记录内容](05-data-collection.md#54)。

## 1.4 支持的平台与依赖版本

对应参考手册的"支持的平台与系统要求"。

| 项 | 要求 |
|---|---|
| 操作系统 | Ubuntu 22.04(已验证);采集路径为 V4L2 + UVC,不支持 macOS / Windows |
| 显卡驱动 | NVIDIA ≥ 570.144 |
| Python | 3.12(v5.1 固定) |
| PyTorch | ≥ 2.2,CUDA 12.8 |
| 夹爪 SDK | `xense.taccap` ≥ 0.1.0(`taccap-gripper` PyPI 包) |
| 环境管理 | 强烈推荐 [Mamba / Miniforge](https://github.com/conda-forge/miniforge)(robostack-staging 通道更快) |
| 视频编解码 | `torchcodec` + `av` wheel(v5.1 不再用 conda 固定 ffmpeg) |

!!! danger "先决条件"
    - 用户需加入 `dialout`、`video` 用户组(见 [3.1 串口权限](03-host-hardware.md#31))。
    - 建议为夹爪串口配置 udev 规则,避免 ModemManager 抢占(见 [3.2](03-host-hardware.md#32))。

下一步 → [2. 环境部署](02-environment.md)
