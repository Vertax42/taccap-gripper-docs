# 7. 常见问题与参考

对应参考手册的"API 手册与用户指南"。

## 7.1 常见问题(FAQ)

常见报错与现场问题已整理到独立的 **[故障排查](troubleshooting.md)** 章(按症状 → 原因 → 解决)。
本页保留配置项、术语表与附录。

## 7.2 `RobotConfig` 常用配置项

| 配置项 | 默认 | 作用 |
|---|---|---|
| `--robot.side` | 自动 | `left`/`right`,两只夹爪都接时必填 |
| `--robot.role` | `leader` | `follower` 绑定 Slave 单元 |
| `--robot.enable_tracker` | `true` | 关闭则只录触觉 + 夹爪 |
| `--robot.tracker_serial` | 未设 | 钉住追踪器 SN,绕过侧别规则 |
| `--robot.enable_wrist_camera` | `true` | 关闭腕相机 |
| `--robot.wrist_camera_width/_height/_fps` | — | 腕相机分辨率/帧率 |
| `--robot.tactile_fps` | — | 触觉帧率 |
| `--robot.tactile_output_types` | — | 触觉输出类型 |
| `--robot.expected_tactiles_per_side` | — | 校验每侧触觉数量 |
| `--robot.gripper_open_rad` | `1.7` | 该单元最大开合角(闭合恒为 0) |
| `enable_init_pose_alignment` | `false` | UMI 式初始位姿对齐(高级选项,默认关闭) |

!!! note "配置源"
    完整字段见 `src/lerobot/robots/taccap_gripper/config_taccap_gripper.py`。

## 7.3 术语表

| 术语 | 含义 |
|---|---|
| **TacCap** | Tactile Capture,触觉采集(夹爪名) |
| **UMI** | Universal Manipulation Interface,手持式主夹爪数采范式 |
| **Leader / Follower** | 主/从;序列号 patch `m`=Master(主),`s`=Slave(从) |
| **单左双右** | 4 位序列号最后一位:奇→左,偶→右 |
| **GSPS** | 视触觉传感器(左右指各一),序列号 `GSPS01...` |
| **XC** | 腕部 UVC 相机,序列号 `XC...` |
| **tcp** | Tool Center Point,末端执行器位姿(`tcp.x/y/z` + 6D 旋转 `r1..r6`) |
| **6D rotation** | `r1..r3` = 旋转矩阵第一列,`r4..r6` = 第二列(同 `vive_tracker`) |
| **shifted-frame** | 移位帧配对:t-1 观测配 t 动作 |
| **self-driven** | 自驱动:设备自身产出观测与演示动作,无独立遥操端 |

## 7.4 本手册范围

!!! note "范围边界"
    本手册只覆盖 **从数采夹爪硬件 → 数据落盘为 `LeRobotDataset`** 的完整链路
    (概述/硬件 → 环境安装 → 软件使用采集 → 数据落盘与介绍)。
    **模型训练、推理、部署不在本手册范围**,请参考对应工程的独立文档。

---

## 参考资料

- 数采主仓库设备说明:`src/lerobot/robots/taccap_gripper/README.md`
- 夹爪 SDK:`third_party/taccap-gripper/`(`README.md` / `docs/ARCHITECTURE.md`)
- 序列号/发现规则:`src/lerobot/robots/taccap_gripper/serial_discovery.py`
