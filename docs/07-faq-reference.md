# 7. 常见问题与参考

## 7.1 常见问题(FAQ)

常见报错与现场问题已整理到独立的 **[故障排查](troubleshooting.md)** 章(按症状 → 原因 → 解决)。
本页保留配置项、术语表与附录。

## 7.2 `RobotConfig` 常用配置项

| 配置项 | 默认 | 作用 |
|---|---|---|
| `robot.side` | 自动 | `left`/`right`,**单夹爪模式**下两只都接着时必填;只接一只则自动选中 |
| `robot.role` | `leader` | 填 `follower` 绑定从夹爪 |
| `robot.enable_tracker` | `true` | 关闭则只录触觉 + 夹爪 |
| `robot.tracker_serial` | 未设 | 钉住追踪器 SN,绕过侧别规则 |
| `robot.enable_wrist_camera` | `true` | 关闭腕相机 |
| `robot.wrist_camera_width/_height/_fps` | — | 腕相机分辨率/帧率 |
| `robot.enable_head_camera` | `false` | 头显相机(第一视角 + 头显位姿),见 [5.6](05-data-collection.md#56) |
| `robot.head_camera_eyes` | `both` | `both` = 左右眼各一个键;`left` / `right` 只录一只 |
| `robot.head_camera_width/_height` | `1024` / `768` | **每只眼**尺寸,只接受 `1024x768` / `1280x960` |
| `robot.head_camera_fps` | `30` | 头显相机帧率 |
| `robot.head_camera_pair_max_skew_ms` | `20.0` | 左右眼帧序号不同时,判为同一次曝光的最大时间差 |
| `robot.head_camera_startup_timeout_s` | `5.0` | connect 时等待首帧的秒数 |
| `robot.head_camera_stale_after_s` | `0.2` | 缓存帧超过该时长即视为过期并告警 |
| `robot.tactile_fps` | `30` | 触觉帧率 |
| `robot.tactile_output_types` | `["rectify"]` | **落盘**的触觉流,**只能填一个**;填多个直接报错 |
| `robot.tactile_display_output_types` | `["difference"]` | **仅供 Rerun 显示**、不落盘的额外触觉流;设为空列表则关闭 |
| `robot.tactile_diff_gain` | `1.0` | `difference` 图的线性增益(只影响显示流);`None` = 用传感器出厂值 |
| `robot.expected_tactiles_per_side` | `2` | 校验每侧触觉数量 |
| `robot.enable_gripper` / `robot.enable_imu` | `true` / `false` | 夹爪本体读数 / IMU 通道 |
| `robot.gripper_open_rad` | `1.7` | **仅从夹爪用**。主夹爪一律用自己固件里实测的行程上限,本项对它没有任何作用——没标定的主夹爪会被拒绝连接,而不是退回这个常量。见 [4.1](04-calibration.md#41) |
| `robot.tracker_to_ee_pos` | `None` | 覆盖 tracker→EE 平移;`None` = 用该侧**内置实测值** |
| `robot.tracker_to_ee_quat` | `None` | 覆盖 tracker→EE 旋转(同上,两者可独立覆盖) |
| `robot.tracker_wait_timeout` | `10.0` | 连接设备时等待追踪器数据的秒数 |

!!! note "完整字段"
    上表是常用项;完整字段以主仓库附带的设备说明为准。

## 7.3 术语表

| 术语 | 含义 |
|---|---|
| **TacCap** | 包名 `xense.taccap` 与设备类型 `taccap_gripper` 里的名字(Tactile Capture);产品名为 XTac-UMI G1 |
| **UMI** | Universal Manipulation Interface,手持式主夹爪数采范式 |
| **Leader / Follower** | 主夹爪 / 从夹爪;序列号 patch `m` = leader,`s` = follower |
| **V2.1**(固件) | 固件的**命令集**版本 —— 实现了哪些命令,不是构建号(构建号是 leader `≥ 1.2.0` / follower `≥ 1.1.0`,更高版本同样支持),也不是帧格式(帧格式为 V1.8)。行程标定命令由 V2.1 引入 → [三套编号](versions.md#v21) |
| **单左双右** | 4 位序列号最后一位:奇→左,偶→右 |
| **GSPS** | 视触觉传感器(左右指各一),序列号 `GSPS01...` |
| **XC** | 腕部 UVC 相机,序列号 `XC...` |
| **tcp** | Tool Center Point,末端执行器位姿(`tcp.x/y/z` + 6D 旋转 `r1..r6`) |
| **6D rotation** | 旋转矩阵 R(世界 ← 本体)的**前两列**:`r1..r3` = 第一列 = 本体 X 轴在世界系下的方向,`r4..r6` = 第二列 = Y 轴;第三列为叉积,可算回 → [约定详解](05-data-collection.md#53) |
| **shifted-frame** | 移位帧配对:t-1 观测配 t 动作 |
| **self-driven** | 自驱动:设备自身产出观测与演示动作,无独立遥操端 |

## 7.4 本手册范围

!!! note "范围边界"
    本手册只覆盖 **从数采夹爪硬件 → 数据落盘为 `LeRobotDataset`** 的完整链路
    (概述/硬件 → 环境安装 → 软件使用采集 → 数据落盘与介绍)。
    **模型训练、推理、部署不在本手册范围**,请参考对应工程的独立文档。

---

## 参考资料

- 数采主仓库 [`xense-taccap-lerobot`](https://github.com/Vertax42/xense-taccap-lerobot) 附带的设备说明
- 夹爪 SDK [`TacCap-Gripper`](https://github.com/Vertax42/TacCap-Gripper)(子模块 `third_party/taccap-gripper/`)
- 追踪器 PC 服务 [`XenseVR-PC-Service`](https://github.com/Vertax42/XenseVR-PC-Service)(子模块 `third_party/XenseVR-PC-Service/`)
- 视触觉传感器 SDK [`xensesdk`](https://github.com/XenseRobotics/xensesdk) · [文档站](https://xensedoc.readthedocs.io/en/latest/)
