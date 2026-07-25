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

## 磁盘规划与估算 {#storage-planning}

- 双夹爪多相机的原始视频数据吞吐可达约 **280 MB/s**;这不是编码后的磁盘写入速度。实时编码后的体积取决于分辨率、画面内容、`fps`、编码器和码率。
- 粗估单条 episode 体积 ≈ 各编码视频流平均码率 × 时长 + Parquet 与元数据。相机数、分辨率、`fps`、`episode_time_s` 都会放大体积。
- 大批量采集前先录制 2–3 条代表性 episode,运行完整性检查,再用 `du -sh` 测量实际体积并按计划集数估算。不要直接用原始 280 MB/s 推算落盘量。

```bash
du -sh ~/.cache/huggingface/lerobot/<你的org>/<数据集名>
df -h ~/.cache/huggingface/lerobot          # 看目标盘剩余空间
```

## 备份与上传

- **本地备份**:重要数据集在删除/迁移前先备份整个 `<repo_id>/` 目录。
- **上传 Hub**:用 `lerobot-push-dataset-to-hub`(见 [6.4 上传 Hugging Face Hub](06-dataset.md#64));
  大数据集加 `--upload-large-folder`,私有加 `--private`。
- 上传即视为一次**异地备份 + 交付**;上传前先 `lerobot-check-dataset` 校验。

## 清理与维护

- 磁盘紧张时,先确认数据已通过 `lerobot-check-dataset` 校验并完成备份 / 上传。为保留恢复机会,优先把目标数据集移动到单独的隔离目录,观察无误后再通过文件管理器删除:

```bash
mkdir -p /data/lerobot-trash
realpath ~/.cache/huggingface/lerobot/<你的org>/<旧数据集名>
du -sh ~/.cache/huggingface/lerobot/<你的org>/<旧数据集名>
mv -- ~/.cache/huggingface/lerobot/<你的org>/<旧数据集名> /data/lerobot-trash/
```

移动前必须核对 `realpath` 输出是预期的单个数据集目录;不要对缓存根目录执行递归删除。

- 定期用 `df -h` 盯住目标盘,避免采集中途写满(见 [故障排查 → 数据与磁盘](troubleshooting.md))。

## 采集台账(建议)

为可追溯,建议为每份数据集记录:

| 记录项 | 示例 |
|---|---|
| `repo_id` | `Xense/pick_object_20260703` |
| 任务描述 | `Pick up the object` |
| 用的夹爪 / 追踪器 SN | `TCGU01A24Z0001m` / `PC2310MLL...` |
| 标定时间 | 2026-07-03 |
| 软件版本 / commit | `xense-taccap-lerobot main@d7b74a6c`;`xense.taccap 0.1.4` |
| 世界系会话 | XenseVR-Toolkit 启动时间 / 操作者朝向 |
| 完整性检查 | `lerobot-check-dataset` 通过;异常 episode 列表 |
| 集数 / 单集时长 | 50 / 15s |
| 备注 | 光照/场景/异常集 |
