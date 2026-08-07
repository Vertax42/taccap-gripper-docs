# 2. 环境部署

对应参考手册的"开发环境部署"。本章面向 Ubuntu 22.04 LTS / Ubuntu 24.04 LTS,
把 `xense-taccap-lerobot` 及全部硬件 SDK 装好并验证。

!!! info "XTac-UMI G1 已验证环境"
    XTac-UMI G1 硬件联调与采集流程在 `mamba` 环境 `xense-taccap` 下验证。当前可读取的测试主机环境:

    - OS: Ubuntu 22.04.5 LTS / Ubuntu 24.04.4 LTS
    - Linux kernel: **6.8 / 6.14 / 7.0 系列均已验证**,内核版本不构成约束
    - 机器架构: `x86_64`
    - Python: `3.12.13`(主仓库要求 **≥ 3.12**)
    - 仓库 commit 与各包版本: 见 [版本与支持](versions.md)(唯一出处,避免多处抄写走样)

    Ubuntu 22.04 LTS 也是本章覆盖的目标环境;其它发行版或架构需按实际驱动、UVC、串口权限和 `.deb` 包支持情况单独验证。

    建议采集主机配 NVIDIA GPU,这样 `--dataset.vcodec=auto` 可使用 GPU H.264 硬件编码器,降低多路视频实时编码时的 CPU 压力。
    **配了 NVIDIA GPU 的话,驱动需 ≥ 570.144**(`nvidia-smi --query-gpu=driver_version --format=csv,noheader` 可查)。

## 选择安装方式 {#choose}

有两条路径,装出来的采集环境是同一套,选一条走完即可。

| | **Mamba 源码安装** | **Docker 交付镜像** |
|---|---|---|
| 拿到的东西 | 源码仓库,自己建环境 | 一个打包好的交付目录,跑一个脚本 |
| 耗时 | 较长,要编译三个硬件 SDK | 十几分钟(主要在导入镜像) |
| 架构 | amd64 / arm64 | **仅 amd64** |
| NVIDIA GPU | 可选(没有 GPU 也能采) | **必需**,驱动 ≥ 570.144 |
| 环境隔离 | 装在主机的 Mamba 环境里 | 装在容器里,不污染主机 |
| 改代码 | 方便 | 不方便 |
| 步骤 | 下面 2.1 – 2.5 | [Docker 交付镜像](#docker) |

**默认走 Mamba**——适用面最广:amd64 / arm64 都支持,没有 NVIDIA GPU 也能采,
要改采集程序也方便。本手册后续章节的命令都按这条路径书写。

机器正好满足 **amd64 + NVIDIA 驱动**、又不想自己建环境时,可以选 Docker。

!!! warning "两条路径都需要能访问外网"
    安装过程要从网上取东西——conda 要拉 conda-forge 与 PyPI 包、克隆 GitHub 仓库与子模块、
    下载 XenseVR PC Service 的 `.deb`;Docker 要装 Docker Engine 与 NVIDIA Container Toolkit。
    **离线机器两条都走不通。**

    在受限网络里,给终端配好代理再开始(Docker 那条脚本支持 `XENSE_PROXY_URL`,见
    [一键安装](#docker))。

以下 2.1 – 2.5 是 **Mamba 源码安装**这条路径。

!!! info "走 Docker 的话,2.1 – 2.5 整段跳过"
    镜像里已经装好了 Mamba 环境、采集程序和三个硬件 SDK,不需要克隆仓库、建环境或跑
    `setup_env.sh`——直接看后面的 [Docker 交付镜像](#docker)。

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
| [`third_party/taccap-gripper`](https://github.com/Vertax42/TacCap-Gripper) | `xense.taccap`(XTac-UMI G1 触觉夹爪 SDK) |
| [`third_party/XenseVR-PC-Service`](https://github.com/Vertax42/XenseVR-PC-Service) | `xensevr_pc_service_sdk`(Pico4 Ultra 企业版追踪器 / 头显相机) |

!!! note "xensesdk 不是子模块"
    `xensesdk` 是视触觉传感器 SDK,由 `setup_env.sh --install` 自动安装,
    无需单独拉取子模块。

!!! danger "更新子模块后必须重新编译 `xense.taccap`"
    `taccap-gripper` 的 Python 包里带一份**编译产物**。`git submodule update` 只更新文件,
    不会重新编译——文件更新了而编译产物还是旧的时候,`import xense.taccap` 会直接失败,例如:

    ```text
    AttributeError: module 'xense.taccap._taccap_native' has no attribute 'GripperAutoCalConfig'
    ```

    拉取任何含 `cpp/` 或 `python/bindings/` 改动的子模块更新后,重新构建:

    ```bash
    cd ~/xense-taccap-lerobot
    LIBRARY_PATH="${CONDA_PREFIX}/lib" \
      uv pip install -e third_party/taccap-gripper --no-deps --no-build-isolation
    ```

    或直接 `bash setup_env.sh --install`(会一并处理其他 SDK)。**不需要 sudo。**
    完成后验证:

    ```bash
    python -c "import xense.taccap as t; print(t.__version__)"
    ```

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
    [v0.2.0 release](https://github.com/Vertax42/XenseVR-PC-Service/releases/tag/v0.2.0)
    下载当前机器架构对应的 `.deb` 包(可用 `$XENSEVR_DEB_URL` 覆盖下载地址),
    再执行 `sudo dpkg -i` 安装;已安装同版本时会跳过。

!!! warning "必须是 v0.2.0 才有头显相机"
    v0.2.0 的改动是**转发 `0x30`(带时间戳的视频帧)消息**给 SDK 客户端,而不再丢弃——
    [头显相机](05-data-collection.md#57)的画面就走这条路。追踪器功能与 v0.1.0 完全一致
    (`0x30` 在 v0.1.0 里没有使用,老版本 APK 对新服务也照常工作),所以**不用头显相机的话,
    这次升级对你没有任何行为变化**。

    **arm64 主机会被固定在 v0.1.0。** v0.2.0 只发布了 amd64 包,`setup_env.sh` 检测到 arm64 时
    会自动退回到最后一个带 arm64 资源的版本并打印说明——代价是 arm64 上没有头显相机。
    确实需要时,在 ARM64 主机上用仓库里的 `RoboticsService/qt-gcc_aarch64.sh` 自行编译。

## 2.5 验证安装 {#25}

三个包全部能 import 即成功:

```bash
python -c 'import xensevr_pc_service_sdk; print("xensevr_pc_service_sdk OK ->", xensevr_pc_service_sdk.__file__)'
python -c 'import xensesdk; print("xensesdk OK ->", xensesdk.__file__)'
python -c 'import xense.taccap; print("xense.taccap OK ->", xense.taccap.__file__)'
```

用[头显相机](05-data-collection.md#57)时再补一条——检查是不是带相机接口的新版本:

```bash
python -c 'import xensevr_pc_service_sdk as xrt; print("pico camera API:", hasattr(xrt, "has_pico_camera_frame"))'
```

返回 `False` 说明环境里加载的还是旧版本,重新执行 `./setup_env.sh --install` 即可。

可选:确认视频编解码依赖可加载(`torchcodec` 按 PyTorch 兼容矩阵固定,PyAV 固定为 `15.1.0`;FFmpeg 不参与 conda 求解):

```bash
python -c 'import torchcodec; print("torchcodec OK ->", torchcodec.__version__)'
python -c 'import av; print("PyAV OK ->", av.__version__)'
```

!!! tip "需要系统 ffmpeg?"
    若你需要带 `libsvtav1` 的系统 ffmpeg,请单独安装(apt 或官方静态构建);
    当前 0.5.1 定制分支的默认编码路径不依赖它。

---

## Docker 交付镜像 {#docker}

镜像里已经包含完整的 `xense-taccap` 环境、CUDA 用户态库、采集程序和三个硬件 SDK
(XenseSDK、TacCap-Gripper、Pico4 绑定),容器启动时还会自动拉起 XenseVR PC Service。

!!! warning "容器以特权模式运行"
    为了支持采集过程中热插拔触觉传感器、腕相机、夹爪串口和 Pico4,容器使用特权模式
    并共享主机网络。**只在你信任的采集主机上运行。**

### 主机要求

- Ubuntu 22.04 / 24.04,**amd64**
- **NVIDIA 驱动 ≥ 570.144** —— `nvidia-smi --query-gpu=driver_version --format=csv,noheader` 可查

安装脚本**不会替你装或升级显卡驱动**(涉及显卡型号、Secure Boot 和重启),驱动不满足
会直接停下来提示。Docker Engine、Compose 插件和 NVIDIA Container Toolkit 缺了则由脚本自动装。

### 一键安装

把交付目录整个拷到采集主机,用**普通用户**(不要用 root)执行:

```bash
cd xense-taccap-lerobot-<版本>-linux-amd64
./install_customer.sh
```

脚本依次完成:检查系统与显卡驱动 → 按需安装 Docker 与 NVIDIA Container Toolkit →
安装夹爪串口的 udev 规则(即 [3.2](03-host-hardware.md#32) 的 ModemManager 屏蔽规则)→
校验镜像 SHA256 并导入 → 跑一次 PyTorch CUDA 冒烟测试。

!!! tip "需要走代理时"
    ```bash
    XENSE_PROXY_URL=http://127.0.0.1:7897 ./install_customer.sh
    ```

### 安装后的主机设置

让当前用户立刻拿到 Docker 权限:

```bash
sudo systemctl enable --now docker
sudo usermod -aG docker "$USER"
newgrp docker        # 或者注销重登
docker images
```

!!! warning "`docker` 组成员权限接近 root"
    只把需要采集的用户加进去,不要图省事把所有本地用户都加上。

要在容器里显示 Rerun 窗口,还需由主机当前**图形桌面用户**授权一次:

```bash
xhost +si:localuser:root
# 采集结束后可撤销
xhost -si:localuser:root
```

### 进入容器与验证

```bash
docker compose run --rm xense-taccap
```

在容器里逐条确认——四个 import 全过、GPU 可见:

```bash
python -c 'import torch; print(torch.__version__, torch.cuda.is_available())'
python -c 'import xensesdk; print("xensesdk ->", xensesdk.__file__)'
python -c 'import xense.taccap; print("taccap ->", xense.taccap.__file__)'
python -c 'import xensevr_pc_service_sdk; print("pico4 ->", xensevr_pc_service_sdk.__file__)'
```

再确认设备能被发现:

```bash
lerobot-find-cameras
lerobot-info
```

不进交互 shell 也可以直接跑一条命令:

```bash
docker compose run --rm xense-taccap lerobot-info
```

!!! note "数据不会随容器删除而丢失"
    数据集根目录在容器里是 `/data/lerobot`,连同 XenseSDK 的传感器配置缓存、
    Hugging Face 与 Torch 缓存都存在 Docker volume 里,`--rm` 删掉临时容器不影响它们。
    查看数据:`docker compose run --rm xense-taccap bash -lc 'ls -la /data'`。

!!! tip "只处理数据、不接 Pico4 时"
    容器默认会启动 XenseVR PC Service。用不到追踪器时可以关掉:

    ```bash
    START_XENSEVR_SERVICE=0 docker compose run --rm xense-taccap
    ```

---

环境装好后,**先做主机与硬件配置**(串口权限、设备发现),否则夹爪能被列出却打不开。

下一步 → [3. 主机与硬件配置](03-host-hardware.md)
