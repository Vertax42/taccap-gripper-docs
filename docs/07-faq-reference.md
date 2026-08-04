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
| `--robot.enable_head_camera` | `false` | 头显相机(第一视角 + 头显位姿),见 [5.7](05-data-collection.md#57) |
| `--robot.head_camera_eyes` | `both` | `both` = 左右眼各一个键;`left` / `right` 只录一只 |
| `--robot.head_camera_width/_height` | `1024` / `768` | **每只眼**尺寸,只接受 `1024x768` / `1280x960` |
| `--robot.head_camera_fps` | `30` | 头显相机帧率 |
| `--robot.head_camera_pair_max_skew_ms` | `20.0` | 左右眼帧序号不同时,判为同一次曝光的最大时间差 |
| `--robot.head_camera_startup_timeout_s` | `5.0` | connect 时等待首帧的秒数 |
| `--robot.head_camera_stale_after_s` | `0.2` | 缓存帧超过该时长即视为过期并告警 |
| `--robot.tactile_fps` | `30` | 触觉帧率 |
| `--robot.tactile_output_types` | `["rectify"]` | **落盘**的触觉流,**只能填一个**;填多个直接报错 |
| `--robot.tactile_display_output_types` | `["difference"]` | **仅供 Rerun 显示**、不落盘的额外触觉流;设为空列表则关闭 |
| `--robot.tactile_diff_gain` | `1.0` | `difference` 图的线性增益(只影响显示流);`None` = 用传感器出厂值 |
| `--robot.expected_tactiles_per_side` | `2` | 校验每侧触觉数量 |
| `--robot.enable_gripper` / `--robot.enable_imu` | `true` / `false` | 夹爪本体读数 / IMU 通道 |
| `--robot.gripper_open_rad` | `1.7` | **仅回退用**的全局常量。夹爪按 [4.1](04-calibration.md#41) 标定后,`gripper.pos` 用的是该台**固件里实测的行程上限**,本项不参与;只有未标定、固件低于 V2.1 或 follower 才会退回除以这个数 |
| `--robot.tracker_to_ee_pos` | `None` | 覆盖 tracker→EE 平移;`None` = 用该侧**内置实测值** |
| `--robot.tracker_to_ee_quat` | `None` | 覆盖 tracker→EE 旋转(同上,两者可独立覆盖) |
| `--robot.tracker_wait_timeout` | `10.0` | connect 时等待追踪器数据的秒数 |

!!! note "配置源"
    完整字段见 `src/lerobot/robots/taccap_gripper/config_taccap_gripper.py`。

## 7.3 术语表

| 术语 | 含义 |
|---|---|
| **TacCap**(代码名) | 代码/包名 `xense.taccap`、`taccap_gripper` 的由来(Tactile Capture);产品显示名为 XTac-UMI G1 |
| **UMI** | Universal Manipulation Interface,手持式主夹爪数采范式 |
| **Leader / Follower** | 主/从;序列号 patch `m`=Master(主),`s`=Slave(从) |
| **单左双右** | 4 位序列号最后一位:奇→左,偶→右 |
| **GSPS** | 视触觉传感器(左右指各一),序列号 `GSPS01...` |
| **XC** | 腕部 UVC 相机,序列号 `XC...` |
| **tcp** | Tool Center Point,末端执行器位姿(`tcp.x/y/z` + 6D 旋转 `r1..r6`) |
| **6D rotation** | `r1..r3` = 旋转矩阵第一列,`r4..r6` = 旋转矩阵第二列 |
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
