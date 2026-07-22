# 2. 环境部署

对应参考手册的"开发环境部署"。本章在 Ubuntu 22.04 上把 `xense-taccap-lerobot`
及全部硬件 SDK 装好并验证。

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
| `third_party/XenseVR-PC-Service` | `xensevr_pc_service_sdk`(Pico4 遥操 / 追踪器) |
| `third_party/XenseVR-RobotVision-PC` | ZED-M → Pico4 立体透视(单独构建) |

!!! note "xensesdk 不是子模块"
    `xensesdk`(视触觉成像)发布在 PyPI(cp312 manylinux wheel,内置打过补丁的
    `libxense_c.so`)。`setup_env.sh --install` 会用 `uv pip install xensesdk`
    直接装。离线/自定义构建可指定本地 wheel:
    ```bash
    export XENSESDK_WHEEL=/path/to/xensesdk-*-cp312-*-linux_x86_64.whl
    ```

## 2.3 创建并激活环境

```bash
bash ./setup_env.sh --mamba lerobot-xense
mamba activate lerobot-xense
```

!!! tip "环境名"
    `conda_environment.yaml` 里默认名就是 `lerobot-xense`。可以传别的名给 `--mamba`,
    但本手册统一默认用 `lerobot-xense`。

## 2.4 一键安装

```bash
bash ./setup_env.sh --install
```

这一步会:

- 用 `conda_environment.yaml` 更新 conda 环境
- 从 `pyproject.toml` 安装主包
- 从 PyPI 安装 `xensesdk`(`uv pip install xensesdk`,可用 `$XENSESDK_WHEEL` 覆盖)
- 安装 **XenseVR PC Service 守护进程**(约 100 MB 的 `.deb`,装到 `/opt/apps/roboticsservice`)
- 构建 `third_party` 下的 SDK:`xensevr_pc_service_sdk`(Pico4)与 `xense.taccap`(夹爪)

!!! note "XenseVR PC Service 的 .deb 从哪来"
    `setup_env.sh --install` 会**优先用本地副本**(`$XENSEVR_DEB`、仓库 `dist/`
    或 `~/Downloads/`),否则从
    [v0.1.0 release](https://github.com/Vertax42/XenseVR-PC-Service/releases/tag/v0.1.0)
    下载对应架构的包(可用 `$XENSEVR_DEB_URL` 覆盖),再 `sudo dpkg -i`(幂等,
    同版本会跳过)。

## 2.5 验证安装

三个包全部能 import 即成功:

```bash
python -c 'import xensevr_pc_service_sdk; print("xensevr_pc_service_sdk OK ->", xensevr_pc_service_sdk.__file__)'
python -c 'import xensesdk; print("xensesdk OK ->", xensesdk.__file__)'
python -c 'import xense.taccap; print("xense.taccap OK ->", xense.taccap.__file__)'
```

可选:确认视频编解码 wheel 可加载(v5.1 用 `torchcodec` + `av`,不再 conda 固定 ffmpeg):

```bash
python -c 'import torchcodec; print("torchcodec OK ->", torchcodec.__version__)'
```

!!! tip "需要系统 ffmpeg?"
    若你需要带 `libsvtav1` 的系统 ffmpeg,请单独安装(apt 或上游静态构建);
    v5.1 的默认编码路径不依赖它。

---

环境装好后,**先做主机与硬件配置**(串口权限、设备发现),否则夹爪能被列出却打不开。

下一步 → [3. 主机与硬件配置](03-host-hardware.md)
