# 数据管理与命名

数据落盘之后怎么组织、命名、估算磁盘、备份与清理。数据集**格式细节**见
[数据集与示例](06-dataset.md);本页讲**运维与规范**。

## 落盘位置

采集默认写到:

```
~/.cache/huggingface/lerobot/<repo_id>/
```

用 `--dataset.root <path>` 可指定其它位置(磁盘更大/更快的盘)。目录结构大致:

```
<repo_id>/
├── data/     # parquet(观测/动作等表格数据)
├── videos/   # mp4(触觉 + 腕相机)
└── meta/     # json/jsonl:info · episodes · stats · tasks
```

字段与格式详见 [6.1 LeRobotDataset 格式速览](06-dataset.md#61)。

## 命名规范(`repo_id`)

`repo_id` 形如 `<org>/<name>`,建议:

- **小写 + 连字符/下划线**,不含空格与中文。
- **含任务与日期**,便于检索与去重:`Xense/pick_object_20260703`。
- **一个数据集对应一个任务/变体**;不同任务或明显不同的变体拆成不同 `repo_id`
  (见 [最佳实践 → 多样性 vs 一致性](best-practices.md))。
- `--dataset.single_task` 的任务描述在同一数据集内保持一致(会写入 `meta/tasks`)。

!!! tip "建议约定(团队内统一)"
    `<org>/<任务>_<变体>_<YYYYMMDD>`,例:`Xense/insert_plug_left_20260703`。

## 磁盘规划与估算

- 满负荷出流可达 **~280 MB/s**(双臂多相机)。采集前务必确认目标盘的**空间与写入带宽**。
- 粗估单条 episode 体积 ≈ 各视频流码率 × 时长 + 表格数据。相机数、分辨率、`fps`、
  `episode_time_s` 都会放大体积。
- 建议采集大批量前先采几条,`du -sh ~/.cache/huggingface/lerobot/<repo_id>` 看实际单集体积,
  再乘以计划集数估算总量。

```bash
du -sh ~/.cache/huggingface/lerobot/<你的org>/<数据集名>
df -h ~/.cache/huggingface/lerobot          # 看目标盘剩余空间
```

## 备份与上传

- **本地备份**:重要数据集在删除/迁移前先备份整个 `<repo_id>/` 目录。
- **上传 Hub**:用 `push_dataset_to_hub.py`(见 [6.4 上传 HuggingFace Hub](06-dataset.md#64));
  大数据集加 `--upload-large-folder`,私有加 `--private`。
- 上传即视为一次**异地备份 + 交付**;上传前先 `lerobot-check-dataset` 校验。

## 清理与维护

- 磁盘紧张时清理**已上传/已备份**的旧数据集释放空间:

```bash
# 确认已备份/已上传后再删
rm -rf ~/.cache/huggingface/lerobot/<你的org>/<旧数据集名>
```

- 定期用 `df -h` 盯住目标盘,避免采集中途写满(见 [故障排查 → 数据与磁盘](troubleshooting.md))。

## 采集台账(建议)

为可追溯,建议为每份数据集记录:

| 记录项 | 示例 |
|---|---|
| `repo_id` | `Xense/pick_object_20260703` |
| 任务描述 | `Pick up the object` |
| 用的夹爪 / 追踪器 SN | `TCGU01A24Z0001m` / `PC2310MLL...` |
| 标定时间 | 2026-07-03 |
| 集数 / 单集时长 | 50 / 15s |
| 备注 | 光照/场景/异常集 |
