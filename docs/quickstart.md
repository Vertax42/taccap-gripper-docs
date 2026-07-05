# 一页速通(TL;DR)

已经装好环境、标定过设备?这一页从**上电到出第一条数据**,照抄即可。第一次用请先走
[环境安装](02-environment.md) 与 [标定与自检](04-calibration.md)。

!!! note "前提"
    - 已完成 [环境安装](02-environment.md)(`setup_env.sh --install` 通过、三个包能 import)。
    - 已做 [串口权限 + ModemManager](03-host-hardware.md#31) 一次性主机配置。
    - 已 [标定编码器零点](04-calibration.md#41)。
    - `mamba activate lerobot-xense`。

## 1. 上电顺序

```mermaid
flowchart LR
    A[插夹爪 USB] --> N[接 Pico4 有线网络<br/>关闭电脑 WiFi] --> B[开头显·配对追踪器] --> C[面朝机器人启动<br/>XenseVR-Toolkit] --> D[启动 XenseVR PC Service]
```

```bash
/opt/apps/roboticsservice/runService.sh    # 启动 XenseVR PC Service
```

!!! danger "数采时关闭电脑 WiFi"
    Pico4 走**有线共享网络**,电脑 WiFi 会与之冲突导致追踪不稳/连不上。数采期间关闭数采电脑 WiFi。
    见 [3.4 网络连接](03-host-hardware.md#pico-network)。

!!! warning "采集全程不要重启 XenseVR-Toolkit"
    重启会重设世界原点,导致同一数据集内位姿参考系不一致。

## 2. 自检(确认设备就绪)

```bash
# 夹爪可读:role 应为 Leader/Follower,firmware_sn 非空
python -c "from xense.taccap import scan_grippers
for g in scan_grippers(): print(g.side.name, g.role.name, repr(g.firmware_sn))"
```

出问题看 [故障排查](troubleshooting.md)。

## 3. 录制一条数据

单臂(单只夹爪自动选中;两只都接时加 `--robot.side=left|right`):

```bash
lerobot-record \
    --robot.type=taccap_gripper \
    --robot.id=right --robot.side=right \
    --dataset.repo_id=<你的org>/<数据集名> \
    --dataset.num_episodes=1 \
    --dataset.episode_time_s=10 \
    --dataset.single_task='Pick up the object'
```

- Pico4 位姿会**自动录制**;只想要触觉+夹爪加 `--robot.enable_tracker=false`。
- 双臂用 `--robot.type=bi_taccap_gripper`。
- 加 `--display_data=true` 开 Rerun 3D 可视化。

细节见 [数据采集](05-data-collection.md)。

## 4. 看数据

```bash
lerobot-check-dataset --repo-id <你的org>/<数据集名>
```

## 5.(可选)上传 Hub

```bash
python push_dataset_to_hub.py \
    --repo-id <你的org>/<数据集名> \
    --dataset-path ~/.cache/huggingface/lerobot/<你的org>/<数据集名> \
    --upload-large-folder
```

数据集长什么样、每帧记录了什么 → [数据集与示例](06-dataset.md)。
