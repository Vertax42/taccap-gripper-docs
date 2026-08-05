# 6. 数据集与示例

对应参考手册的"SDK 示例"。本章讲采完之后:数据集长什么样、如何校验、回放、上传 Hub。

!!! abstract "第一次用 lerobot?先读官方文档"
    我们的数据落盘为标准 **LeRobotDataset v3.0** 格式。没接触过 lerobot 的用户,强烈建议先读官方文档,
    再回来看本章的 XTac-UMI G1 具体用法:

    - :material-book-open-variant: **LeRobotDataset v3.0 官方文档**:<https://huggingface.co/docs/lerobot/lerobot-dataset-v3>
      (格式设计、目录结构、录制、加载/流式、v2.1→v3.0 迁移)
    - :material-book-open-variant: lerobot 录制指南:<https://huggingface.co/docs/lerobot/il_robots#record-a-dataset>
    - :material-book-open-variant: lerobot 文档首页:<https://huggingface.co/docs/lerobot>

    **v3.0 要点**:基于文件存储(多集合并进大 Parquet/MP4 文件)、关系型元数据定位集边界、
    支持 Hub 原生流式;相比 v2 大幅减少小文件、初始化更快。

## 6.1 LeRobotDataset 格式速览 {#61}

采集产出标准 `LeRobotDataset`,默认落在 `~/.cache/huggingface/lerobot/<repo_id>/`。
它可以像普通 Hugging Face / PyTorch 数据集一样索引:

```python
from lerobot.datasets.lerobot_dataset import LeRobotDataset

ds = LeRobotDataset("<your_org>/<your_dataset>")
sample = ds[0]          # 单帧:观测 + 动作,均为 torch tensor
```

序列化方式:

- `hf_dataset`:Hugging Face datasets → parquet
- 视频(触觉 + 腕相机):mp4(省空间)
- 元数据:纯 json / jsonl(`info` / `episodes` / `stats` / `tasks`)

!!! tip "时序查询 delta_timestamps"
    可按与索引帧的时间关系一次取多帧,如
    `delta_timestamps = {"observation.image": [-1, -0.5, -0.2, 0]}` 取当前帧及其前
    1s / 0.5s / 0.2s 三帧。

`info` 里的关键元数据:`fps`、`features`、`total_episodes`、`total_frames`、
`robot_type`、`data_path`、`video_path`。

## 6.2 数据校验 {#62}

用 `lerobot_check_dataset.py` 检查数据集完整性(帧数、视频、字段一致性等):

```bash
# 从仓库脚本运行
python src/lerobot/scripts/lerobot_check_dataset.py \
    --repo-id Xense/assemble_box_with_phone_stand \
    --root ~/.cache/huggingface/lerobot

# 或用已安装的控制台入口
lerobot-check-dataset --repo-id Xense/assemble_box_with_phone_stand \
    --root ~/.cache/huggingface/lerobot

# 只查某几集
lerobot-check-dataset --repo-id Xense/assemble_box_with_phone_stand --episode-index 0 2 4
```

| 参数 | 含义 |
|---|---|
| `--repo-id` | 数据集仓库 id(`<org>/<name>`) |
| `--root` | 本地根目录(默认 `~/.cache/huggingface/lerobot`) |
| `--episode-index` | 只检查指定集(可多值,如 `0 2 4`) |

!!! note "脚本来源"
    `lerobot_check_dataset.py` 随主仓库一起提供,以你本地这一版为准。

## 6.3 回放与可视化

- **在线数据集可视化**:数据集上传到 Hugging Face Hub 后,可使用 [LeRobot Dataset Visualizer](https://huggingface.co/spaces/lerobot/visualize_dataset) 在线浏览各 episode 的视频与数据。
- **3D 轨迹**:采集/自检时加 `--display_data=true`,Rerun 中查看夹爪位姿与轨迹
  (见 [4.4](04-calibration.md#44))。
- **数据集浏览**:用 lerobot 官方的数据集可视化工具打开 parquet + mp4 逐帧检查
  (以你本地这一版 `lerobot` 提供的可视化脚本为准)。

## 6.4 上传 Hugging Face Hub {#64}

用已安装的 `lerobot-push-dataset-to-hub` 控制台入口推送到 Hub:

```bash
# 基本用法(需要 --repo-id 与 --dataset-path)
lerobot-push-dataset-to-hub \
    --repo-id Vertax/xense_flare_pick_and_place \
    --dataset-path ~/.cache/huggingface/lerobot/Vertax/xense_flare_pick_and_place
```

常用变体:

=== "大数据集"

    ```bash
    lerobot-push-dataset-to-hub \
        --repo-id Xense/assemble_box_with_phone_stand0410 \
        --dataset-path ~/.cache/huggingface/lerobot/Xense/assemble_box_with_phone_stand0410 \
        --upload-large-folder
    ```

=== "私有仓库"

    ```bash
    lerobot-push-dataset-to-hub \
        --repo-id Vertax/xense_flare_pick_and_place \
        --dataset-path ~/.cache/huggingface/lerobot/Vertax/xense_flare_pick_and_place \
        --private
    ```

=== "不传视频"

    ```bash
    lerobot-push-dataset-to-hub \
        --repo-id Vertax/xense_flare_pick_and_place \
        --dataset-path ~/.cache/huggingface/lerobot/Vertax/xense_flare_pick_and_place \
        --no-videos
    ```

成功后数据集地址为 `https://huggingface.co/datasets/<repo_id>`。

!!! tip "先登录 Hub"
    上传前确保已执行 `hf auth login`(旧版也可使用 `huggingface-cli login`),或设置 `HF_TOKEN`,否则会因鉴权失败。

## 6.5 完整命令示例合集

先录制到本地并校验;确认完整后再按需上传 Hub。以下命令均显式指定 30 FPS、120 秒录制、60 秒复位和不自动上传。

=== "单夹爪(以右侧为例)"

    ```bash
    lerobot-record \
        --robot.type=taccap_gripper \
        --robot.side=right \
        --display_data=true \
        --dataset.repo_id=Xense/pick_object_demo \
        --dataset.single_task='Pick up the object' \
        --dataset.num_episodes=20 \
        --dataset.fps=30 \
        --dataset.episode_time_s=120 \
        --dataset.reset_time_s=60 \
        --dataset.push_to_hub=false
    ```

=== "双夹爪"

    ```bash
    lerobot-record \
        --robot.type=bi_taccap_gripper \
        --display_data=true \
        --dataset.repo_id=Xense/pick_object_demo \
        --dataset.single_task='Pick up the object' \
        --dataset.num_episodes=20 \
        --dataset.fps=30 \
        --dataset.episode_time_s=120 \
        --dataset.reset_time_s=60 \
        --dataset.push_to_hub=false
    ```

录制完成后执行:

```bash
# 1) 检查本地数据完整性
lerobot-check-dataset --repo-id Xense/pick_object_demo

# 2) 可选:上传 Hub
lerobot-push-dataset-to-hub \
    --repo-id Xense/pick_object_demo \
    --dataset-path ~/.cache/huggingface/lerobot/Xense/pick_object_demo \
    --upload-large-folder
```

下一步 → [7. 常见问题与参考](07-faq-reference.md)
