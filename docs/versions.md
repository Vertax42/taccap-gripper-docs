# 版本与支持

本手册对应的版本基线、如何查版本、如何升级,以及遇到问题怎么反馈。

## 版本兼容基线

本页将“支持范围”和“已验证基线”分开列出。实际命令与字段仍以本地 checkout 为准。

| 组件 | 支持范围 / 约束 | 已验证基线 |
|---|---|---|
| 操作系统 | Ubuntu 22.04 / 24.04 | Ubuntu 24.04.4 LTS |
| NVIDIA GPU / 驱动 | GPU 可选;多路视频建议使用 NVIDIA H.264 硬件编码 | 驱动 570.144 |
| Python | ≥ 3.10 | 3.12.13 |
| PyTorch | 由 `setup_env.sh` 与依赖锁定文件统一安装 | 2.10.0 |
| `xense-taccap-lerobot` | 基于 lerobot 0.5.1 定制 | `main@d7b74a6c` |
| `xense.taccap`(`taccap-gripper` SDK) | 与主仓库子模块版本配套 | 0.1.4 |
| 夹爪固件协议 | 帧格式 V1.8 + 命令集 V1.7 | V1.8 / V1.7 |
| `xensesdk` | 由安装脚本提供 | 2.1.1 |
| `xensevr_pc_service_sdk` | 随 XenseVR PC Service `.deb` | v0.1.0 release |

!!! note "版本以本地为准"
    命令与字段应以你 checkout 的 `src/lerobot/robots/taccap_gripper/README.md` 与
    `third_party/taccap-gripper/` 为准。

## 如何查版本

```bash
# xense-taccap / SDK
python -c "import xense.taccap as t; print('xense.taccap', t.__version__)"
python -c "import xensesdk, xensevr_pc_service_sdk; print('xensesdk/pc_service OK')"

# 固件 SN(含固件方案信息;role/side 由 SN 解析)
python -c "from xense.taccap import scan_grippers
for g in scan_grippers(): print(g.side.name, g.role.name, repr(g.firmware_sn))"

# 视频编解码
python -c "import torchcodec; print('torchcodec', torchcodec.__version__)"
```

## 升级与更新

- **仓库 + 子模块**:
  ```bash
  git pull --recurse-submodules
  git submodule update --init --recursive --progress
  ./setup_env.sh --install     # 重新对齐依赖
  ```
- **固件 OTA**:通过 SDK 的 `OtaSession` / `ota_update.py`,见 [SDK 示例](sdk-examples.md)。

!!! danger "OTA 有风险"
    刷错固件会**变砖 MCU**。核对目标固件与设备型号后再操作。

## 支持与反馈

遇到问题:

1. 先查 [故障排查](troubleshooting.md) 与 [常见问题](07-faq-reference.md)。
2. 仍未解决,通过**内部渠道反馈**(*待补:issue 地址 / 对接人 / 群*)。

反馈时请附带:

- **完整报错**(不要截断)。
- 自检输出:`scan_grippers` 的 side / role / firmware_sn。
- 版本信息:上面「如何查版本」的输出。
- 复现步骤:用的命令、单/双臂、是否接追踪器等。

## 待补

- 内部 issue / 工单地址与对接人
- 固件 ↔ SDK ↔ lerobot 的精确兼容矩阵(跨版本)
- 发布节奏 / 变更日志入口
