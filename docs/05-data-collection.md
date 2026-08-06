# 5. 数据采集

对应参考手册的"SDK 使用"。本章是核心:用 `lerobot-record` 采集并写出 `LeRobotDataset`。

## 5.1 采集原理

`taccap_gripper` **不需要遥操作端**,录制是自驱动的。

数据采用**移位帧(shifted-frame)配对**:动作比观测**领先一步**——*t-1* 步的观测,配
*t* 步的位姿(EEF TCP 位姿 + 归一化 `gripper.pos`,开了[头显相机](#57)时还包括
**头显位姿**)作为动作。因此每集会少 1 帧
(它没有前一帧可配对)。集与集之间的复位阶段是被动等待:重新摆放设备即可,无需遥操。

!!! note "命令行上没有 --teleop.*"
    正因为自驱动,录制命令里**不出现任何 `--teleop.*` 参数**。

## 开录前:先用 `lerobot-teleoperate` 看一眼 {#preview}

`lerobot-teleoperate` **只读取并预览设备,不写任何数据**,跑一遍是零成本的;录制则是有成本的
——一条集录废了要重来。而多数问题(某路相机没上来、追踪器没位姿、`gripper.pos` 顶不到 1.0、
双臂左右接反)在预览里一眼就能看出来。

和 [4.3 端到端冒烟测试](04-calibration.md#43) 不同,这里**所有设备都开着**,要确认的正是
整条链路是否凑齐。

```bash
lerobot-teleoperate \
    --robot.type=bi_taccap_gripper \
    --robot.enable_head_camera=false \
    --fps=30 \
    --display_data=true \
    --show_trajectory=true
```

**单夹爪**:换成 `--robot.type=taccap_gripper` 并加 `--robot.side=left|right`,其余相同。

在 Rerun 里逐项确认:

| 看什么 | 期望 |
|---|---|
| 左右两路触觉 | 都有画面;按压时纹理明显变化 |
| 腕相机 | 有画面,视野里没有线缆或杂物遮挡 |
| `gripper.pos` | 张到底到 **1.0**、闭合到 **0.0**——顶不到 1.0 见 [4.1.3](04-calibration.md#413) |
| `/world` 里的 EE 标记与轨迹 | 随夹爪平滑移动,不跳变、不卡住——**追踪器要始终在头显视野内**,被遮挡就会丢跟踪 |
| 双臂:左右两条轨迹 | 各自独立且对应正确的手,没有接反 |

都正常再 `Ctrl+C` 退出,继续下面的录制。用头显相机时把
`--robot.enable_head_camera` 改成 `true` 一并预览(见 [§5.7](#57))。

## 5.2 双夹爪录制

设备**按序列号规则自动发现**——不列举夹爪/触觉/相机序列号。触觉、腕相机、追踪器都按同一套
规则各自匹配左右。

```bash
lerobot-record \
    --robot.type=bi_taccap_gripper \
    --robot.enable_tracker=true \
    --robot.enable_head_camera=false \
    --display_data=true \
    --dataset.repo_id=<your_org>/<your_dataset> \
    --dataset.num_episodes=1 \
    --dataset.fps=30 \
    --dataset.push_to_hub=false \
    --dataset.episode_time_s=120 \
    --dataset.reset_time_s=60 \
    --dataset.single_task='Pick up the object'
```

### 参数详解 {#params}

`lerobot-record` 参数分三类:**数据集**(`--dataset.*`)、**录制控制**(顶层)、
**设备**(`--robot.*`)。完整定义参考 lerobot 官方
[录制指南](https://huggingface.co/docs/lerobot/v0.5.1/en/il_robots#record-a-dataset)(对应本项目定制的 lerobot 基线 0.5.1)。

#### 数据集参数 `--dataset.*`

| 参数 | 默认 | 含义 |
|---|---|---|
| `repo_id` | **必填** | 数据集标识,`{HF用户名}/{数据集名}`,如 `Xense/pick_demo` |
| `single_task` | **必填** | 任务的简短准确描述,写入 `meta/tasks`(如 `'Pick up the object'`) |
| `root` | `$HF_LEROBOT_HOME/repo_id` | 本地存储目录;不指定则用默认缓存路径 |
| `fps` | `30` | 采样(录制)帧率上限 |
| `episode_time_s` | `120` | 每集录制时长(秒) |
| `reset_time_s` | `60` | 每集之间的复位时长(秒),被动等待你重新摆放场景 |
| `num_episodes` | `50` | 录制集数 |
| `video` | `true` | 是否把帧编码为视频(mp4) |
| `push_to_hub` | `false` | 默认仅保存到本地;需要上传时显式设为 `true` |
| `private` | `false` | 上传为 Hub 私有仓库 |
| `tags` | 无 | 给 Hub 数据集打标签 |
| `streaming_encoding` | `true` | 实时流式编码(见 [§5.5](#55)) |
| `vcodec` | `auto` | 视频编码器(`h264`/`hevc`/`libsvtav1`/`auto`/硬件编码器) |
| `encoder_threads` | 自动 | 每个编码器实例的线程数 |
| `encoder_queue_maxsize` | `30` | 每相机缓冲帧数(~1s@30fps),编码跟不上时反压丢旧帧 |
| `video_encoding_batch_size` | `1` | 批量编码前累计的集数(1=即时编码) |

!!! note "显式写出关键参数"
    推荐命令始终显式指定 `fps=30`、`episode_time_s=120`、`reset_time_s=60` 和 `push_to_hub=false`,避免不同 checkout 的默认值变化影响采集。
    上面的示例把 `--robot.enable_head_camera=false` 也写了出来,同样是这个道理——它默认就是
    `false`,写出来是让这个开关在命令里可见。改成 `true` 即连同头显双目画面与头显位姿一起录
    (见 [§5.7](#57));需要 **PC Service ≥ v0.2.0 且主机为 amd64**,arm64 上保持 `false`。

!!! note "`fps` 与传感器帧率"
    `fps` 是**录制采样率**,不是传感器上限。视触觉传感器本身 120 Hz([硬件参数](hardware.md#specs)),
    按需以更低 `fps` 录制是**使用选择**,不改变传感器规格。

#### 录制控制(顶层参数)

| 参数 | 默认 | 含义 |
|---|---|---|
| `--robot.type` | **必填** | `taccap_gripper`(单夹爪)/ `bi_taccap_gripper`(双夹爪) |
| `--fps` | `30` | **主循环**帧率(设备读取与预览)。与 `--dataset.fps`(落盘采样率)是两个参数,通常设成相同值 |
| `--display_data` | `false` | 在 Rerun 中显示相机画面与 3D 视图 |
| `--show_trajectory` | `true` | Rerun 中叠加 3D 位姿 + 轨迹(需 `display_data` 且有 `tcp.*`) |
| `--display_compressed_images` | `false` | Rerun 里是否 JPEG 压缩后再显示。**默认关**——压缩发生在录制主循环上,开着会吃掉大量帧预算;只有 Rerun 查看器在另一台机器上(`--display_ip`)时才划算 |
| `--display_image_every_n` | `1` | 每 N 帧才刷新一次相机画面(标量始终全速)。**最后手段**,只在仍然超时才动它——它是唯一会改变操作员所见内容的选项 |
| `--play_sounds` | `true` | 语音播报录制事件 |
| `--resume` | `false` | 在已有数据集上**续录** |
| `--teleop.*` | — | 遥操作端;**XTac-UMI G1 自驱动无需**,不用填 |

#### 设备参数 `--robot.*`(XTac-UMI G1 专属)

| 参数 | 默认 | 含义 |
|---|---|---|
| `--robot.side` | 自动 | `left`/`right`,两只夹爪都接时必填 |
| `--robot.role` | `leader` | 填 `follower` 绑定从夹爪 |
| `--robot.enable_tracker` | `true` | 关闭则只录触觉 + 夹爪(无位姿) |
| `--robot.tracker_serial` | 未设 | 钉住追踪器 SN,绕过侧别自动匹配 |
| `--robot.enable_wrist_camera` | `true` | 关闭腕相机 |
| `--robot.wrist_camera_width/_height/_fps` | — | 腕相机分辨率 / 帧率 |
| `--robot.enable_head_camera` | `false` | 录制 Pico4 Ultra 企业版**头显相机**,见 [§5.7](#57) |
| `--robot.head_camera_eyes` | `both` | `both` 录左右两只眼(两个键),`left` / `right` 只录一只 |
| `--robot.head_camera_width/_height` | `1024` / `768` | **每只眼**的尺寸,只接受 `1024x768` 或 `1280x960` |
| `--robot.head_camera_fps` | `30` | 头显相机录制帧率 |
| `--robot.head_camera_pair_max_skew_ms` | `20.0` | 左右眼帧序号不同时,判定为同一次曝光的最大时间差 |
| `--robot.tactile_fps` | `30` | 触觉录制帧率 |
| `--robot.tactile_output_types` | `["rectify"]` | **落盘**的触觉流,**只能填一个** |
| `--robot.tactile_display_output_types` | `["difference"]` | **仅显示**、不落盘的额外触觉流 |
| `--robot.tactile_diff_gain` | `1.0` | `difference` 图的增益(只影响显示) |
| `--robot.expected_tactiles_per_side` | — | 校验每侧触觉数量 |

Pico4 Ultra 企业版追踪器上电后,6-DoF 位姿**自动录制**——追踪器按序列号末尾字母 `G` 前一个数字(单左双右)
自动匹配本单元侧别。

!!! tip "只录触觉 + 夹爪"
    加 `--robot.enable_tracker=false` 关闭位姿录制。

!!! tip "追踪器序列号不合规 / PC 服务枚举不稳"
    用 `--robot.tracker_serial=<SN>` 直接钉住序列号——**逐字使用**,不枚举、不校验
    (打错会在 connect 时报设备找不到)。留空(默认)则走自动发现。

## 5.3 单夹爪录制

只录一只时用 `--robot.type=taccap_gripper`,其余参数同上。单只夹爪自动选中;两只都接入时
用 `--robot.side=left|right` 指定录哪一只。

```bash
lerobot-record \
    --robot.type=taccap_gripper \
    --robot.side=right \
    --robot.enable_tracker=true \
    --robot.enable_head_camera=false \
    --display_data=true \
    --dataset.repo_id=<your_org>/<your_dataset> \
    --dataset.num_episodes=1 \
    --dataset.fps=30 \
    --dataset.push_to_hub=false \
    --dataset.episode_time_s=120 \
    --dataset.reset_time_s=60 \
    --dataset.single_task='Pick up the object'
```

## 5.4 每帧记录内容 {#54}

| Key | 来源 | 形状 / 类型 |
|---|---|---|
| `tcp.x`, `tcp.y`, `tcp.z` | 追踪器位姿 → 经安装变换得到的 **EEF TCP** | float(米) |
| `tcp.r1`..`tcp.r6` | EE 的 6-D 旋转 | float |
| `gripper.pos` | XTac-UMI G1 编码器,归一化 | float ∈ [0, 1] |
| `imu.accel.{x,y,z}`(默认关) | XTac-UMI G1 IMU | float(m/s²) |
| `imu.gyro.{x,y,z}`(默认关) | XTac-UMI G1 IMU | float(rad/s) |
| `imu.mag.{x,y,z}`(默认关) | XTac-UMI G1 IMU | float(µT) |
| `tactile_left` / `tactile_right` | 视触觉校正图 | uint8,约 `(400, 700, 3)` |
| `wrist_cam` | 腕部相机 | uint8 `(H, W, 3)` |
| `left_head` / `right_head`(默认关) | 头显相机,**一只眼一个键** | uint8,默认 `(768, 1024, 3)` |
| `head_camera.x/y/z`(默认关) | 头显位置,与 `tcp.*` 同一世界系;**同时也是动作** | float(米) |
| `head_camera.r1..r6`(默认关) | 头显姿态的 6-D 旋转;**同时也是动作** | float |

!!! note "6-D 旋转约定"
    `r1..r3` 是旋转矩阵第一列,`r4..r6` 是旋转矩阵第二列。

!!! info "`tcp.*` 记的是夹爪末端,不是追踪器"
    追踪器拧在夹爪手柄上,它自己的位姿离**两指中点**约 195 mm。落盘前会乘上一个内置的
    刚性安装变换(取自 CAD 装配实测,左右各一套),所以 `tcp.*` 是 **EEF TCP 位姿**。

    该变换是**机体固连**的——随夹爪一起转,任意姿态下都成立,采集时朝哪个方向起手都行。
    TCP 取两指中点,对称张合时该点不动,因此与 `gripper.pos` 无关。
    想直观确认,把夹爪平放,看 Rerun `/world` 里的 EE 标记是否落在两指中点
    → [4.4 3D 轨迹可视化](04-calibration.md#44)。

!!! tip "IMU 默认不采集"
    上表的 `imu.*` 共 9 个通道**默认关闭**,加 `--robot.enable_imu=true` 才会记录
    (双臂同样是这个开关,两侧一起生效,键名带 `left_` / `right_` 前缀)。

    开启后 `observation.state` 维度相应增加:单夹爪 10 → 19,双夹爪 20 → 38。
    不需要惯性数据就保持关闭,可少写 9 列。

**观测键调节**:

- **触觉** → `tactile_left` / `tactile_right`;校正图为横向 `(400,700,3)`(宽高自动推导,
  **别写死**)。用 `--robot.tactile_fps` / `--robot.tactile_output_types` 调;
  `--robot.expected_tactiles_per_side` 校验每侧数量。

!!! danger "落盘的是 `rectify`,不是你在 Rerun 里看到的那张图"
    两路触觉流**故意不同**:

    - **落盘** = `--robot.tactile_output_types`,默认 `rectify` ——**未做基线相减**的原图,
      保留传感器看到的全部信息。**只能填一个类型**(每个传感器对应数据集里一个视频键),
      填多个会直接报错并提示改用显示流。
    - **显示** = `--robot.tactile_display_output_types`,默认 `difference` ——相对传感器
      **初始化时刻基线**的增强差分图。这张图接触更易读,所以 Rerun 里给操作员看的是它;
      键名形如 `tactile_left_difference`,**不在** `observation_features` 里,不会落盘。

    差分图是**破坏性**的:基线在传感器 init 时抓取,所以**连接时压在胶上的任何力都会被
    整段采集减掉**。这就是它只用于显示、不进数据集的原因——不要为了"看着清楚"把
    `--robot.tactile_output_types` 改成 `difference`。

    `--robot.tactile_diff_gain`(默认 `1.0`)只影响显示流的增益。传感器出厂值 1.5 在本胶体上
    噪声偏大且会削顶;它同时放大信号与噪声,**不改变信噪比**,只是留出余量。
- **腕相机** → `wrist_cam`;`--robot.enable_wrist_camera=false` 跳过;
  `--robot.wrist_camera_width/_height/_fps` 调。
- **头显相机** → `left_head` / `right_head` + `head_camera.*`;**默认关闭**,
  `--robot.enable_head_camera=true` 开启,详见 [§5.7](#57)。
- **角色** → `--robot.role=follower` 绑定从夹爪(默认 `leader`)。

## 5.5 录制选项:流式编码与编码器预热 {#55}

视频键(触觉 + 腕相机)**在采集时实时编码**,而非先存 PNG 再在集尾编码,因此
**每集结束时几乎不用等**。默认开启(`--dataset.streaming_encoding=true`):

```bash
lerobot-record \
    --robot.type=taccap_gripper --robot.side=right \
    --dataset.repo_id=<your_org>/<your_dataset> \
    --dataset.num_episodes=20 \
    --dataset.fps=30 \
    --dataset.push_to_hub=false \
    --dataset.reset_time_s=60 \
    --dataset.episode_time_s=120 \
    --dataset.single_task='Pick up the object' \
    --dataset.streaming_encoding=true \
    --dataset.encoder_threads=2 \
    --dataset.vcodec=auto
```

- 每个相机一个 `_CameraEncoderThread`,通过有界队列喂原始帧
  (`--dataset.encoder_queue_maxsize`,约 1 秒帧量);编码器跟不上时**丢弃最旧帧并告警**,
  不阻塞采集循环。
- `--dataset.vcodec=auto` 会优先启用可用的硬件编码。推荐采集主机配 NVIDIA GPU,
  这样可使用 GPU H.264 硬件编码器,降低多路视频实时编码时的 CPU 压力;无 NVIDIA GPU 时仍可录制,
  但高分辨率或多相机场景更容易出现编码跟不上。

!!! note "编码器预热"
    编码器初始化约 25 ms,拖到第一帧才做会让首帧严重超出 `fps` 预算。因此每集录制前会
    **先把编码器预热好**,等全部就绪再开录——首帧不再付初始化开销。

## 5.6 分集与复位

- 一次运行采多集:`--dataset.num_episodes=N`。
- 集与集之间是**被动复位**:重新摆放设备,无需遥操。
- 录制过程中可用 lerobot 的键盘控制(重录当前集、提前结束等,按 `lerobot-record` 的通用约定)。

!!! tip "想采到"好数据"?"
    会跑命令只是第一步。务必阅读 [采集规范与最佳实践](best-practices.md)——坐标原点纪律、
    触觉接触、演示一致性、增量验证等,直接决定落盘数据的质量。

## 5.7 可选:头显相机(第一视角) {#57}

**默认关闭。**打开后录制 Pico4 Ultra 企业版**头显自带的双目相机**,以及头显自身的位姿——
也就是操作员的第一视角画面和"人在往哪看"。单夹爪(`taccap_gripper`)和双夹爪
(`bi_taccap_gripper`)都支持,开关和参数完全一样。

```bash
lerobot-record \
    --robot.type=bi_taccap_gripper \
    --robot.enable_tracker=true \
    --robot.enable_head_camera=true \
    --display_data=true \
    --dataset.repo_id=<your_org>/<your_dataset> \
    --dataset.single_task='Pick up the object' \
    --dataset.fps=30 \
    --dataset.push_to_hub=false
```

产出三组键:

| Key | 含义 |
|---|---|
| `left_head` / `right_head` | 头显相机画面,**一只眼一个视频键**,默认各 `(768, 1024, 3)` |
| `head_camera.x/y/z` | 头显位置(米);**同时进 action** |
| `head_camera.r1..r6` | 头显姿态,6-D 旋转(约定同 `tcp.*`);**同时进 action** |

!!! warning "`left_` / `right_` 在这里指的是**眼睛**,不是左右手"
    双夹爪上 `{side}_wrist`、`{side}_tcp.*` 是按**手臂**分左右的;但头显只有一个,
    `left_head` / `right_head` 指的是**头显的左眼 / 右眼**。同理:两个单夹爪进程同时开头显相机,
    拿到的是**同一个头显的同一路画面**,不是两个独立视角。

### 前置条件

1. **XenseVR PC Service ≥ v0.2.0**(amd64)。v0.1.0 会丢弃承载相机帧的 `0x30` 消息;
   arm64 主机目前固定在 v0.1.0,即**没有头显相机**。见 [2.4 一键安装](02-environment.md)。
2. **头显 APP 正在推流**。相机和追踪器**共用同一条 SDK 连接**,所以头显必须已连上 PC Service
   (见 [3.5 启动 XenseVR PC Service](03-host-hardware.md#35))。反过来,关掉相机不会断开追踪器的
   连接,关掉追踪器也不会断开相机。

### 分辨率与录制单眼

`--robot.head_camera_width/_height` **只接受 `1024x768`(默认)和 `1280x960`**,填别的直接报错,
不会悄悄降级;首帧尺寸与配置不一致时同样报错——重采样会**悄悄改掉记录下来的视场角**。
两个尺寸都是 4:3,与传感器一致(PICO 的相机访问接口单帧上限 2328x1748,也是 4:3;
按 16:9 要画面,得到的是裁剪或拉伸,而不是更大的视场)。

只要一只眼时用 `--robot.head_camera_eyes=left`(或 `right`):JPEG 解码量和编码器压力都减半,
数据集里也只有一个头部视频键。

!!! danger "改分辨率或改录制的眼睛 = 换了一组数据"
    这两项一变,落盘的画面就不一样了,**变更前后的 episode 不能混用**。要改就在开录之前定下来,
    并在[数据管理](data-management.md)里记清楚。

### 左右眼配对

两只眼是**两条独立消息**分别到达的,而且分成两个视频键之后,一旦配错在数据里**看不出任何痕迹**。
所以每帧都会比对两只眼的最新帧:帧序号相同就是确定的同一次曝光;序号不同则要求时间戳相差不超过
`--robot.head_camera_pair_max_skew_ms`(默认 20 ms,对比 30 fps 下约 33 ms 的帧周期)。
超出不会中断录制,而是打一条**限流的告警并给出实测偏差**——让问题看得见,而不是默默写进数据集。

!!! note "为什么要在后台线程里收帧"
    直接在录制循环里读 SDK,实测有 **7% 的帧**拿到的是"一只眼更新了、另一只没更新"的组合
    (599 次采样,左眼总是快 1–2 帧)。现在改为后台线程以 120 Hz 轮询,凑齐一对才交出去,
    实测 7% → 0%,帧率不受影响。

### 位姿与可视化

`head_camera.*` 是**头显位姿**,并且已经重映射到与 `tcp.*` **相同的重力对齐世界系**
(用的是追踪器那套 Pico→world 变换)。这意味着头和手第一次可以直接比较、画在同一个 3D 场景里:
开 `--display_data=true` 时,`/world` 视图里会多出一个琥珀色的 `HEAD` 标记
(见 [4.4](04-calibration.md#44))。

!!! note "头显位姿既是观测,也是动作"
    `head_camera.*` 同时写进 `observation` 和 `action`,和 `tcp.*` 一样参与
    [移位帧配对](#51)——**演示时操作员看向哪里,本身就是演示的一部分**,是策略要复现的目标,
    而不只是背景信息。位置 + 6-D 旋转的排布也和 `tcp.*` 完全一致,可以同样地被指令。

    双臂上它**不带 `left_` / `right_` 前缀,而且只出现一次**——两条臂共用一个头显,不像
    `{side}_tcp.*` 那样左右各记一份。

    落盘的图像**不进 action**,只在观测里。

!!! note "维度变化"
    开启后 `observation.state` 增加 9 维(头显位姿)。单夹爪 10 → 19,双夹爪 20 → 29。
    再叠加 `--robot.enable_imu=true` 时按各自规则继续累加。

下一步 → [采集规范与最佳实践](best-practices.md) → [数据集与示例](06-dataset.md)
