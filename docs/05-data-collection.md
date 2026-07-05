# 5. 数据采集

对应参考手册的"SDK 使用"。本章是核心:用 `lerobot-record` 采集并写出 `LeRobotDataset`。

## 5.1 采集原理

`taccap_gripper` **不需要遥操作端**(`RecordConfig.__post_init__` 允许它
`teleop=None`)。录制由 `lerobot_record.py` 里专门的 `self_driven_record_loop`
处理(通过 `SELF_DRIVEN_RECORD_ROBOTS` 路由)。

**移位帧(shifted-frame)配对**:每条记录用 *t-1* 步的观测,配 *t* 步的位姿
(Pico4 位姿 + 归一化 `gripper.pos`)作为动作——动作**领先观测一步**,是真正的
"移动到下一步"目标,而非退化的同帧位姿。每集丢弃 1 帧(首帧没有前驱)。集与集之间的
复位阶段是被动等待:重新摆放设备即可,无需遥操。

!!! note "命令行上没有 --teleop.*"
    正因为自驱动,录制命令里**不出现任何 `--teleop.*` 参数**。

## 5.2 单臂录制

设备**按序列号规则自动发现**——不列举夹爪/触觉/相机序列号。单只夹爪自动选中;两只都
接入时用 `--robot.side=left|right` 指定。

```bash
lerobot-record \
    --robot.type=taccap_gripper \
    --robot.id=right \
    --robot.side=right \
    --dataset.repo_id=<your_org>/<your_dataset> \
    --dataset.num_episodes=1 \
    --dataset.episode_time_s=10 \
    --dataset.single_task='Pick up the object'
```

### 参数详解 {#params}

`lerobot-record` 参数分三类:**数据集**(`--dataset.*`)、**录制控制**(顶层)、
**设备**(`--robot.*`)。完整定义参考 lerobot 官方
[录制指南](https://huggingface.co/docs/lerobot/il_robots#record-a-dataset)。

#### 数据集参数 `--dataset.*`

| 参数 | 默认 | 含义 |
|---|---|---|
| `repo_id` | **必填** | 数据集标识,`{HF用户名}/{数据集名}`,如 `Xense/pick_demo` |
| `single_task` | **必填** | 任务的简短准确描述,写入 `meta/tasks`(如 `'Pick up the object'`) |
| `root` | `$HF_LEROBOT_HOME/repo_id` | 本地存储目录;不指定则用默认缓存路径 |
| `fps` | `30` | 采样(录制)帧率上限 |
| `episode_time_s` | `60` | 每集录制时长(秒) |
| `reset_time_s` | `60` | 每集之间的复位时长(秒),被动等待你重新摆放场景 |
| `num_episodes` | `50` | 录制集数 |
| `video` | `true` | 是否把帧编码为视频(mp4) |
| `push_to_hub` | `true` | ⚠️ **默认会上传到 HuggingFace Hub**;仅本地保存请设 `false` |
| `private` | `false` | 上传为 Hub 私有仓库 |
| `tags` | 无 | 给 Hub 数据集打标签 |
| `streaming_encoding` | `true` | 实时流式编码(见 [§5.5](#55)) |
| `vcodec` | `auto` | 视频编码器(`h264`/`hevc`/`libsvtav1`/`auto`/硬件编码器) |
| `encoder_threads` | 自动 | 每个编码器实例的线程数 |
| `encoder_queue_maxsize` | `30` | 每相机缓冲帧数(~1s@30fps),编码跟不上时反压丢旧帧 |
| `video_encoding_batch_size` | `1` | 批量编码前累计的集数(1=即时编码) |

!!! note "`fps` 与传感器帧率"
    `fps` 是**录制采样率**,不是传感器上限。视触觉传感器本身 120 Hz([硬件参数](hardware.md#specs)),
    按需以更低 `fps` 录制是**使用选择**,不改变传感器规格。

#### 录制控制(顶层参数)

| 参数 | 默认 | 含义 |
|---|---|---|
| `--robot.type` | **必填** | `taccap_gripper`(单臂)/ `bi_taccap_gripper`(双臂) |
| `--display_data` | `false` | 在 Rerun 中显示相机画面与 3D 视图 |
| `--show_trajectory` | `true` | Rerun 中叠加 3D 位姿 + 轨迹(需 `display_data` 且有 `tcp.*`) |
| `--display_compressed_images` | `true` | Rerun 用 JPEG 显示以降内存;要无损设 `false` |
| `--play_sounds` | `true` | 语音播报录制事件 |
| `--resume` | `false` | 在已有数据集上**续录** |
| `--teleop.*` | — | 遥操作端;**TacCap 自驱动无需**,不用填 |

#### 设备参数 `--robot.*`(TacCap 专属)

| 参数 | 默认 | 含义 |
|---|---|---|
| `--robot.id` | — | 该单元标识(写入数据集元信息) |
| `--robot.side` | 自动 | `left`/`right`,两只夹爪都接时必填 |
| `--robot.role` | `leader` | `follower` 绑定从爪(Slave) |
| `--robot.enable_tracker` | `true` | 关闭则只录触觉 + 夹爪(无位姿) |
| `--robot.tracker_serial` | 未设 | 钉住追踪器 SN,绕过侧别自动匹配 |
| `--robot.enable_wrist_camera` | `true` | 关闭腕相机 |
| `--robot.wrist_camera_width/_height/_fps` | — | 腕相机分辨率 / 帧率 |
| `--robot.tactile_fps` | — | 触觉录制帧率 |
| `--robot.tactile_output_types` | — | 触觉输出类型 |
| `--robot.expected_tactiles_per_side` | — | 校验每侧触觉数量 |

Pico4 追踪器上电后,6-DoF 位姿**自动录制**——追踪器按序列号倒数第二位(奇左偶右)
自动匹配本单元侧别。

!!! tip "只录触觉 + 夹爪"
    加 `--robot.enable_tracker=false` 关闭位姿录制。

!!! tip "追踪器序列号不合规 / PC 服务枚举不稳"
    用 `--robot.tracker_serial=<SN>` 直接钉住序列号——**逐字使用**,不枚举、不校验
    (打错会在 connect 时报设备找不到)。留空(默认)则走自动发现。

## 5.3 双臂录制

用 `--robot.type=bi_taccap_gripper`(左右两只夹爪同时录),其余参数同上。触觉、腕相机、
追踪器均按同一套规则各自匹配左右。

## 5.4 每帧记录内容 {#54}

| Key | 来源 | 形状 / 类型 |
|---|---|---|
| `tcp.x`, `tcp.y`, `tcp.z` | Pico4 追踪器 → EE | float(米) |
| `tcp.r1`..`tcp.r6` | EE 的 6-D 旋转 | float |
| `gripper.pos` | TacCap 编码器,归一化 | float ∈ [0, 1] |
| `imu.accel.{x,y,z}`(可选) | TacCap IMU | float(m/s²) |
| `imu.gyro.{x,y,z}`(可选) | TacCap IMU | float(rad/s) |
| `imu.mag.{x,y,z}`(可选) | TacCap IMU | float(µT) |
| `tactile_left` / `tactile_right` | 视触觉校正图 | uint8,约 `(400, 700, 3)` |
| `wrist_cam` | 腕部相机 | uint8 `(H, W, 3)` |

!!! note "6-D 旋转约定"
    与 `vive_tracker` 一致:`r1..r3` 是旋转矩阵第一列,`r4..r6` 是第二列。

**观测键调节**:

- **触觉** → `tactile_left` / `tactile_right`;校正图为横向 `(400,700,3)`(宽高自动推导,
  **别写死**)。用 `--robot.tactile_fps` / `--robot.tactile_output_types` 调;
  `--robot.expected_tactiles_per_side` 校验每侧数量。
- **腕相机** → `wrist_cam`;`--robot.enable_wrist_camera=false` 跳过;
  `--robot.wrist_camera_width/_height/_fps` 调。
- **角色** → `--robot.role=follower` 绑定 Slave 单元(默认 `leader`)。

## 5.5 录制选项:流式编码与编码器预热 {#55}

视频键(触觉 + 腕相机)**在采集时实时编码**,而非先存 PNG 再在集尾编码,因此
`save_episode()` 近乎瞬时。默认开启(`--dataset.streaming_encoding=true`):

```bash
lerobot-record \
    --robot.type=taccap_gripper --robot.side=right \
    --dataset.repo_id=<your_org>/<your_dataset> \
    --dataset.num_episodes=20 \
    --dataset.episode_time_s=15 \
    --dataset.single_task='Pick up the object' \
    --dataset.streaming_encoding=true \
    --dataset.encoder_threads=2 \
    --dataset.vcodec=auto
```

- 每个相机一个 `_CameraEncoderThread`,通过有界队列喂原始帧
  (`--dataset.encoder_queue_maxsize`,约 1 秒帧量);编码器跟不上时**丢弃最旧帧并告警**,
  不阻塞采集循环。
- `--dataset.vcodec=auto` 启用硬件编码。

!!! note "编码器预热"
    打开 PyAV 容器 + 编解码上下文约 25 ms,惰性到首帧才做会让首帧严重超出 `fps` 预算。
    因此每集录制前会先 `prepare_episode_recording()` 预热编码器线程并按各视频键声明的
    `(H, W, C)` 打开编解码上下文,阻塞到全部就绪——首帧不再付初始化开销。

## 5.6 分集与复位

- 一次运行采多集:`--dataset.num_episodes=N`。
- 集与集之间是**被动复位**:重新摆放设备,无需遥操。
- 录制过程中可用 lerobot 的键盘控制(重录当前集、提前结束等,按上游 `lerobot-record` 约定)。

!!! tip "想采到"好数据"?"
    会跑命令只是第一步。务必阅读 [采集规范与最佳实践](best-practices.md)——坐标原点纪律、
    触觉接触、演示一致性、增量验证等,直接决定落盘数据的质量。

下一步 → [采集规范与最佳实践](best-practices.md) → [数据集与示例](06-dataset.md)
