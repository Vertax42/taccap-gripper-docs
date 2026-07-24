# 2. 环境部署

对应参考手册的"开发环境部署"。本章面向 Ubuntu 22.04 LTS / Ubuntu 24.04 LTS,
把 `xense-taccap-lerobot` 及全部硬件 SDK 装好并验证。

!!! info "XTac-UMI G1 已验证环境"
    XTac-UMI G1 硬件联调与采集流程在 `mamba` 环境 `xense-taccap` 下验证。当前可读取的测试主机环境:

    - OS: Ubuntu 24.04.4 LTS (Noble Numbat)
    - Linux kernel: `7.0.0-28-generic`(测试机 `uname -r` 输出,非最低内核要求)
    - 机器架构: `x86_64`
    - Python: `3.12.13`
    - 测试仓库: `xense-taccap-lerobot` `main@d7b74a6c`
    - 关键包: `lerobot 0.5.1`, `xense.taccap 0.1.4`, `xensesdk 2.1.1`, `torch 2.10.0`, `torchcodec 0.10.0`, `av 15.1.0`

    Ubuntu 22.04 LTS 也是本章覆盖的目标环境;其它发行版或架构需按实际驱动、UVC、串口权限和 `.deb` 包支持情况单独验证。

    建议采集主机配 NVIDIA GPU,这样 `--dataset.vcodec=auto` 可使用 GPU H.264 硬件编码器,降低多路视频实时编码时的 CPU 压力。

!!! info "总览"
    四步:装 Mamba → 克隆仓库(含子模块)→ 建环境 → `setup_env.sh --install` → 验证。

## 2.1 前置:安装 Mamba / Miniforge

强烈推荐用 Mamba:依赖求解比原生 conda **快约 10×**,且在所有 channel 上都更快。

```bash
curl -L -O "https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-$(uname)-$(uname -m).sh"
bash Miniforge3-$(uname)-$(uname -m).sh
```

## 2.2 克隆仓库与子模块

仓库用 `third_party/` 子模块管理硬件 SDK,**必须**递归克隆:

```bash
git clone \
  --recurse-submodules \
  https://github.com/Vertax42/xense-taccap-lerobot.git
cd xense-taccap-lerobot
```

若已经克隆但漏了子模块:

```bash
git submodule update --init --recursive --progress
```

子模块与对应安装包:

| 子模块 | 安装后的包 |
|---|---|
| `third_party/taccap-gripper` | `xense.taccap`(XTac-UMI G1 触觉夹爪 SDK) |
| `third_party/XenseVR-PC-Service` | `xensevr_pc_service_sdk`(Pico4 Ultra 企业版遥操 / 追踪器) |
| `third_party/XenseVR-RobotVision-PC` | ZED-M → Pico4 Ultra 企业版立体透视(单独构建) |

!!! note "xensesdk 不是子模块"
    `xensesdk` 是视触觉传感器 SDK,由 `setup_env.sh --install` 自动安装,
    无需单独拉取子模块。

## 2.3 创建并激活环境

```bash
./setup_env.sh --mamba
mamba activate xense-taccap
```

!!! tip "环境名"
    `--mamba` 默认创建 `xense-taccap` 环境;
    如需自定义环境名,可在 `--mamba` 后追加名称。

## 2.4 一键安装

```bash
./setup_env.sh --install
```

这一步会:

- 用 `conda_environment.yaml` 更新 conda/mamba 环境
- 从 `pyproject.toml` 安装主包
- 安装 `xensesdk` 视触觉传感器 SDK
- 安装 **XenseVR PC Service 守护进程**(约 100 MB 的 `.deb`,装到 `/opt/apps/roboticsservice`)
- 构建 `third_party` 下的 SDK:`xensevr_pc_service_sdk`(Pico4 Ultra 企业版)与 `xense.taccap`(夹爪)

!!! note "XenseVR PC Service 的 .deb 从哪来"
    `./setup_env.sh --install` 会直接从
    [v0.1.0 release](https://github.com/Vertax42/XenseVR-PC-Service/releases/tag/v0.1.0)
    下载当前机器架构对应的 `.deb` 包(可用 `$XENSEVR_DEB_URL` 覆盖下载地址),
    再执行 `sudo dpkg -i` 安装;已安装同版本时会跳过。

## 2.5 验证安装

三个包全部能 import 即成功:

```bash
python -c 'import xensevr_pc_service_sdk; print("xensevr_pc_service_sdk OK ->", xensevr_pc_service_sdk.__file__)'
python -c 'import xensesdk; print("xensesdk OK ->", xensesdk.__file__)'
python -c 'import xense.taccap; print("xense.taccap OK ->", xense.taccap.__file__)'
```

可选:确认视频编解码依赖可加载(`torchcodec` 按 PyTorch 兼容矩阵固定,PyAV 固定为 `15.1.0`;FFmpeg 不参与 conda 求解):

```bash
python -c 'import torchcodec; print("torchcodec OK ->", torchcodec.__version__)'
python -c 'import av; print("PyAV OK ->", av.__version__)'
```

!!! tip "需要系统 ffmpeg?"
    若你需要带 `libsvtav1` 的系统 ffmpeg,请单独安装(apt 或上游静态构建);
    v5.1 的默认编码路径不依赖它。

---

环境装好后,**先做主机与硬件配置**(串口权限、设备发现),否则夹爪能被列出却打不开。

下一步 → [3. 主机与硬件配置](03-host-hardware.md)
