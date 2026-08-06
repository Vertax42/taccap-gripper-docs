# 版本与支持

本手册对应的版本基线、如何查版本、如何升级,以及遇到问题怎么反馈。

## 必须升级到最新版本 {#required}

!!! danger "这不是可选项——采集前请先把四项都升到下表版本"
    整条链路是**配套**的:固件带[命令集 V2.1](#v21) 才支持行程标定,SDK 0.1.7 才能安全刷这版固件,
    而 `gripper.pos` 的 0–1 归一化要固件、SDK、标定三者齐了才成立。主夹爪缺标定或固件过低时
    **采集程序会直接拒绝连接**,不会让你采出刻度错的数据;但版本不配套的其它组合仍可能
    **"正常"跑完并落盘**,只是数据和别人对不上——事后从数据里看不出来。

| 组件 | 最低要求版本 | 怎么查 |
|---|---|---|
| `xense-taccap-lerobot` | `0.5.1+xtac.0.0.3` | `pip show lerobot` 或看 `pyproject.toml` |
| `xense.taccap` SDK | **0.1.7** | `python -c "import xense.taccap as t; print(t.__version__)"` |
| 夹爪固件 | **命令集 V2.1**,即构建 leader **≥ 1.2.0** / follower **≥ 1.1.0**([区别](#v21)) | 跑 [`calibrate.py`](04-calibration.md#41),版本不够会打印当前版本并退出 |
| 每台 leader 的编码器标定 | 零点 + 行程上限已写入 flash | [4.1 夹爪标定](04-calibration.md#41) |

### 三套编号:V2.1 是命令集,不是固件版本 {#v21}

夹爪固件涉及**三套互不相干的编号**,长得像但含义完全不同:

| 编号 | 当前值 | 是什么 |
|---|---|---|
| 帧格式(wire framing) | `V1.8` | 字节怎么打包成帧(字节填充、CRC)。极少变动,本手册基本不提 |
| **命令集(command set)** | **`V2.1`** | 固件**实现了哪些命令**。每条命令按引入时的版本标注 |
| 固件构建号 | leader `≥ 1.2.0`<br>follower `≥ 1.1.0` | 具体那一版镜像。**这是最小值,不是必须精确匹配的值**——1.2.1 之类的更高版本同样支持 V2.1,不需要为此回刷 |

**本手册里说的「V2.1」一律指命令集**,因为行程标定用的 `EncoderMaxCal` 命令正是 V2.1 引入的
(`V2.0` 引入鱼眼标定,`V1.9` 引入 LED 与私有电机参数)。所以"标定要求固件 ≥ V2.1"说的是
**命令集**要够新。

对应关系是**门槛**而非等号:**命令集 V2.1 从 leader `1.2.0` / follower `1.1.0` 起开始支持**,
更高的构建号(如 leader `1.2.1`)同样支持。所以看到自己是 1.2.1 不必疑惑,也不需要回刷到
1.2.0——**"我们跑的版本"和"我们要求的版本"是两件事**。主从两个号不同,只是因为它们本来就是
两份不同的固件,不是谁更新。

一条命令查完前三项:

```bash
python - <<'EOF'
import xense.taccap as t
from xense.taccap import scan_grippers
print("xense.taccap", t.__version__, "(需要 >= 0.1.7)")
for g in scan_grippers():
    print(f"  {g.firmware_sn}  role={g.role.name}")
EOF
```

**升级顺序不能乱**,每一步都依赖上一步:

```mermaid
flowchart LR
    A[拉仓库 + 子模块] --> B[重新编译 xense.taccap] --> C[刷固件 OTA] --> D[每台 leader 标定]
```

1. [拉仓库与子模块](#repo-update) —— 子模块要跟着一起更新。
2. **重新编译 SDK**,否则新旧文件不配套,`import xense.taccap` 直接失败。
3. [固件 OTA 升级](#ota) —— **必须在第 2 步之后**,理由见该节。
4. [夹爪标定](04-calibration.md#41) —— 升级本身**不产生标定值**,不标的主夹爪连不上。

!!! note "已经采过的数据不用重采"
    这里要求的是**今后采集**用统一版本。历史数据保持原样即可;若要和升级后的数据混用,
    先确认两批的 `gripper.pos` 是否落在同一刻度上(升级+标定前后不一定一致)。

## 版本兼容基线

本页将“支持范围”和“已验证基线”分开列出。实际命令与字段仍以本地 checkout 为准。

| 组件 | 支持范围 / 约束 | 已验证基线 |
|---|---|---|
| 操作系统 | Ubuntu 22.04 / 24.04 | Ubuntu 22.04.5 LTS / 24.04.4 LTS |
| Linux 内核 | 不构成约束 | 6.8 / 6.14 / 7.0 系列均已验证 |
| NVIDIA GPU / 驱动 | GPU 可选;**装了 NVIDIA GPU 则驱动需 ≥ 570.144**。多路视频建议使用 NVIDIA H.264 硬件编码 | 驱动 570.144 / 580.126.09 |
| Python | **≥ 3.12**(`pyproject.toml` 的 `requires-python`;`conda_environment.yaml` 固定 `python=3.12`) | 3.12.13 |
| PyTorch | `torch>=2.2.1,<2.11.0`;`torchvision>=0.21.0,<0.26.0` | 2.10.0 / torchvision 0.25.0 |
| `torchcodec` | `>=0.2.1,<0.11.0`,由 `setup_env.sh` **按当前 torch 版本自动对齐**(不匹配会强制重装) | 0.10.0 |
| PyAV | `av>=15.0.0,<16.0.0`,安装脚本固定装 **15.1.0** | 15.1.0 |
| `rerun-sdk` | `>=0.24.0,<0.27.0`(`--display_data` 用) | 0.26.2 |
| `opencv-python` | 固定 `==4.12.0.88`(XenseRobotics 各 SDK 统一) | 4.12.0.88 |
| NumPy | `>=1.26.4` | 2.2.6 |
| `xense-taccap-lerobot` | 基于 lerobot 0.5.1 定制;版本号 `0.5.1+xtac.0.0.3`(与文档版本同步) | `main@f491cae5` |
| `xense.taccap`(`taccap-gripper` SDK) | 与主仓库子模块版本配套 | 0.1.7(`16412dc`) |
| 夹爪固件命令集 | **V2.1**(帧格式另计,为 V1.8;区别见[三套编号](#v21)) | 命令集 V2.1 |
| 夹爪固件构建 | leader **≥ 1.2.0** / follower **≥ 1.1.0** 即支持命令集 V2.1 | 随 SDK 附带的镜像版本(固件源码分支 `hw_v1.1.0`);具体版本以 SDK 的 `firmware/manifest.json` 为准,见 [固件 OTA 升级](#ota) |
| `xensesdk` | 由安装脚本提供 | 2.1.1 |
| XenseVR PC Service(`.deb` 守护进程) | amd64 ≥ **v0.2.0**;arm64 目前只有 v0.1.0 | v0.2.0(子模块 `6c5ff61d`)|
| `xensevr_pc_service_sdk`(Python 接口) | 随主仓库子模块一起编译安装 | 0.1.0 —— **包版本号没跟着服务走**,见下 |

!!! warning "`pip show xensevr-pc-service-sdk` 会显示 0.1.0,这是正常的"
    这个包的版本号一直是 `0.1.0`,**没有随 PC Service 的 v0.2.0 提升**。所以
    **不要用它判断有没有头显相机支持**——要看的是有没有相机接口:

    ```bash
    python -c "import xensevr_pc_service_sdk as xrt; print(hasattr(xrt, 'has_pico_camera_frame'))"
    ```

    服务本体的版本用 `dpkg -s xensevr-pc-service` 查(见[如何查版本](#check-versions))。

!!! note "PC Service v0.2.0 只影响头显相机"
    v0.2.0 相对 v0.1.0 只多做一件事:**转发头显相机的视频帧**,这是
    [头显相机](05-data-collection.md#57)画面的通路。老版本头显 APK 配 v0.2.0 也照常工作,
    所以**不用头显相机就没有任何行为变化**。

    v0.2.0 只发布了 amd64 包:`setup_env.sh` 在 arm64 上会自动退回 v0.1.0 并打印说明
    (代价是 arm64 没有头显相机),需要时用仓库的 `RoboticsService/qt-gcc_aarch64.sh` 自行编译。

!!! note "版本以本地为准"
    命令与字段应以你本地这一版主仓库、以及夹爪 SDK 附带的设备说明为准。

!!! note "头显相机需要 `ffc94d53` 之后的版本"
    本手册中的[头显相机](05-data-collection.md#57)、Insight 链路移除、以及
    [4.4](04-calibration.md#44) 里 Rerun 视图的改动,来自
    [PR #9](https://github.com/Vertax42/xense-taccap-lerobot/pull/9),已合入 `main`(`ffc94d53`)。

    checkout 停在更早的版本时:`--robot.enable_head_camera` 在单夹爪上还不存在,
    双夹爪上它指的是旧的 Insight 相机,Rerun 里也仍然画着 TRACKER 坐标系和虚线。
    执行 `git pull --recurse-submodules` + `./setup_env.sh --install` 即与本手册一致。

!!! warning "子模块已改用 HTTPS——`ffc94d53` 及更早的版本需要 GitHub SSH key"
    早期版本的 `.gitmodules` 用的是 `git@github.com:` 形式,机器上没配 GitHub SSH key 时
    [克隆仓库与子模块](02-environment.md) 那一步会失败(顶层仓库能克隆,拉子模块时报错)。
    现在已全部改为 `https://`,任何机器都能直接拉。Docker 路径同样要过这一关——镜像构建
    只校验子模块是否已就位,拉取仍由宿主机在构建前完成。

    **已经升级到新版本、但拉子模块仍然要 SSH key**:`.gitmodules` 只是模板,老机器
    `.git/config` 里记的还是旧 URL,`git pull` 不会改写它。同步一次即可:

    ```bash
    git submodule sync --recursive
    git submodule update --init --recursive --progress
    ```

    **必须停在旧版本时**,全局改写 URL 绕过:

    ```bash
    git config --global url."https://github.com/".insteadOf "git@github.com:"
    ```

!!! note "主夹爪拒绝连接的行为需要 `4fb5b79b` 之后的版本"
    更早的版本遇到未标定的主夹爪是**告警后退回** `gripper_open_rad` 常量,会话照常跑完——
    也就是说**能采出刻度错的数据而不报错**。停在旧版本时,务必按 [4.1.3](04-calibration.md#41)
    自己确认 `gripper.pos` 张到底能到 `1.0`。

!!! note "头显位姿进 action、显示默认不压缩,需要 `f491cae5` 之后的版本"
    两项变化都在这之后:`head_camera.*` 从"仅观测"变成**同时也是动作**
    (见 [5.7 位姿与可视化](05-data-collection.md#57));`--display_compressed_images`
    的默认值从 `true` 改为 **`false`**,并新增 `--display_image_every_n`。

    停在更早的版本时:头显位姿只作为观测落盘,策略不会被要求复现它;Rerun 显示默认走 JPEG
    压缩,开 `--display_data=true` 更容易出现 `[slow_frame]`。

!!! note "Docker 交付镜像需要 `9387ef05` 之后的版本"
    [2. 环境部署](02-environment.md#docker) 里那条 Docker 路径——交付目录、
    `install_customer.sh`、`compose.yaml`——来自这次提交,更早的版本里没有。

## 如何查版本 {#check-versions}

一条命令把上表里跑在 Python 环境里的都打出来:

```bash
python - <<'EOF'
import importlib.metadata as M
for p in ("lerobot", "taccap-gripper", "xensesdk", "torch", "torchvision",
          "torchcodec", "av", "rerun-sdk", "opencv-python", "numpy"):
    try:
        print(f"{p:16} {M.version(p)}")
    except M.PackageNotFoundError:
        print(f"{p:16} 未安装")
EOF
```

```bash
# xense-taccap / SDK
python -c "import xense.taccap as t; print('xense.taccap', t.__version__)"
python -c "import xensesdk, xensevr_pc_service_sdk; print('xensesdk/pc_service OK')"

# NVIDIA 驱动(装了 NVIDIA GPU 时需 >= 570.144)
nvidia-smi --query-gpu=driver_version,name --format=csv,noheader

# XenseVR PC Service 守护进程的 deb 版本(服务本体的版本以这个为准)
dpkg -s xensevr-pc-service 2>/dev/null | grep -E '^(Package|Version|Architecture):'
# 是否带头显相机接口(需要 PC Service v0.2.0;包版本号不会变,只能看接口)
python -c "import xensevr_pc_service_sdk as xrt; print('pico camera API:', hasattr(xrt, 'has_pico_camera_frame'))"

# 固件 SN(含固件方案信息;role/side 由 SN 解析)
python -c "from xense.taccap import scan_grippers
for g in scan_grippers(): print(g.side.name, g.role.name, repr(g.firmware_sn))"

# 视频编解码
python -c "import torchcodec; print('torchcodec', torchcodec.__version__)"
```

## 升级与更新

### 仓库 + 子模块 {#repo-update}

```bash
git pull --recurse-submodules
git submodule update --init --recursive --progress
./setup_env.sh --install     # 重新对齐依赖
```

!!! danger "拉完子模块必须重新编译 `xense.taccap`"
    `git submodule update` 只更新文件,不会重新编译。见
    [2.2 克隆仓库与子模块](02-environment.md)。

### 固件 OTA 升级 {#ota}

### 我需要刷吗? {#ota-when}

**不用自己查版本号——跑一次 [`calibrate.py`](04-calibration.md#41) 就知道。**它在写任何东西
之前先验固件,不够会原样退出并打印这台当前的版本:

```text
✗ encoder-max calibration needs firmware >= V2.1 (leader 1.2.0); this gripper reports 1.1.0.0.
  Nothing was changed. Flash it first: ...
```

看到以下**任意一条**就需要刷:

| 现象 | 出处 |
|---|---|
| `calibrate.py` 报 `needs firmware >= V2.1` 并原样退出 | [4.1 夹爪标定](04-calibration.md#41) |
| 主夹爪连不上,报错里提示先做 OTA 升级 | [4.1.1](04-calibration.md#41) |
| 夹爪固件低于命令集 **V2.1**(即 leader < 1.2.0 / follower < 1.1.0) | 上面的[基线表](#版本兼容基线) |

都没遇到就**不用刷**。固件不会自己退化,刷过一次之后除非换主板或擦除固件,不需要再刷。

### 怎么刷

**所有夹爪都要升到 V2.1**。低于 V2.1 的固件不支持行程标定,
[夹爪标定](04-calibration.md#41)的第 2 步做不了,主夹爪也就无法连接。

SDK 自 0.1.7 起**随仓库附带已发布的固件镜像**,直接刷即可:

| 镜像 | 适用角色 |
|---|---|
| `tc-gu-01-master.bin` | 主夹爪(SN 末位 **`m`**) |
| `tc-gu-01-slave.bin` | 从夹爪(SN 末位 **`s`**) |

路径 `third_party/taccap-gripper/firmware/`,该目录只保留当前发布版。

!!! note "镜像版本随 SDK 版本走,以 `manifest.json` 为准"
    **这里不写死版本号**——附带的镜像版本取决于你这一版 SDK。同目录的 `manifest.json`
    记录了每个镜像的版本、字节数与 CRC32,是唯一出处:

    ```bash
    cat third_party/taccap-gripper/firmware/manifest.json
    ```

    只要不低于 leader `1.2.0` / follower `1.1.0`,就已经支持命令集 V2.1。

!!! warning "顺序:**先升级 SDK,再刷固件**"
    0.1.7 之前的 SDK 有两个升级相关的缺陷:固件拒绝写入时它识别不出来,**失败的升级会报成功**;
    重试的方式也可能把一次本来正常的升级弄坏。两者都在 0.1.7 修好了。

    新 SDK 与旧固件通信不变,所以**先升 SDK 总是安全的**。

**按角色选镜像,不是按左右手。**角色看固件 SN 的**最后一个字符**:
`TCGU01A28Z0023m` → `m` → 主夹爪,用 `tc-gu-01-master.bin`。同一套双臂设备上两只夹爪常常**都是主夹爪**。

```bash
# 1. 确认每只夹爪的角色
python -c "from xense.taccap import scan_grippers
for g in scan_grippers(): print(g.firmware_sn, '->', 'master' if g.firmware_sn.endswith('m') else 'slave')"

# 2. 刷写(镜像只写文件名即可,脚本会去 SDK 的 firmware/ 里找)
python third_party/taccap-gripper/python/examples/ota_update.py \
    tc-gu-01-master.bin --side left

# 3. 确认:GetVersion 返回固件编译进去的常量,读回的版本就是实际刷上去的版本
python -c "from xense.taccap import scan_grippers
for g in scan_grippers(): print(g.firmware_sn, g.role.name)"
```

!!! tip "`--target-version` 是可选的"
    它只是给固件的安装后校验日志和分区元数据打个标记,**不影响刷什么内容**——刷进去的是哪一版
    由镜像文件本身决定。需要在日志里标注时才加,填 `manifest.json` 里那个版本号即可:

    ```bash
    python third_party/taccap-gripper/python/examples/ota_update.py \
        tc-gu-01-master.bin --side left --target-version 1.2.1
    ```

约 1 秒写完,夹爪重启并重新识别约 1–3 秒。新固件写在**备用分区**,校验通过之前不会覆盖
正在运行的那一份——所以传输失败不会把夹爪刷坏。

!!! danger "刷错角色会导致夹爪无法启动,需返厂恢复"
    `ota_update.py` 会按 CRC32 与 `manifest.json` 比对识别镜像,**角色不匹配时直接拒绝**
    (不是告警),`--force` 才能强制。手工编译的镜像识别不出来,会带一条提示放行。

    升级期间**不要断电或拔线**(此时夹爪指示灯蓝色闪烁,见 [硬件介绍](hardware.md))。

!!! note "镜像只写文件名,不用管在哪个目录运行"
    `ota_update.py` 会依次尝试:你给的原始路径 → SDK 根目录下的同名路径 →
    SDK `firmware/` 下的同名文件。所以 `tc-gu-01-master.bin` 在主仓库根目录、
    SDK 目录里、以及其它任何位置都能解析到同一个镜像,不需要按仓库拼前缀。
    路径在**连接设备之前**就检查,写错文件名不会白等一次设备发现。

升到 V2.1 之后,回到 [4.1 夹爪标定](04-calibration.md#41) 把零点和行程上限标上——
升级本身不产生标定值,标定完成之前主夹爪仍然连不上。

## 支持与反馈 {#support}

遇到问题:

1. 先查[故障排查](troubleshooting.md)与[常见问题](07-faq-reference.md)。
2. 文档内容、链接或示例问题可提交到[文档仓库 Issues](https://github.com/XenseRobotics/XTac-UMI-G1-Docs/issues)。
3. 硬件、固件、标定材料或返修问题请通过设备交付 / 售后渠道反馈,并提供设备 SN。

反馈时请附带:

- 完整报错与相关日志,不要截断。
- `scan_grippers` 的 side / role / firmware_sn 输出。
- 本页“如何查版本”命令的输出。
- 复现步骤、完整命令、单夹爪 / 双夹爪、是否启用追踪器。
- 如涉及相机或硬件装配,附设备连接和异常画面照片。

## 兼容性与发布维护

- 当前站点文档版本为 `v0.0.3`;内容变更可通过文档仓库 Git 提交历史追踪。
- 主仓库版本号与本页文档版本对齐:`xense-taccap-lerobot` 的 `pyproject.toml` 记 `0.5.1+xtac.0.0.3`,其中 `0.5.1` 是 lerobot 官方基线,`xtac.0.0.3` 是与本文档同步的产品版本。
- 精确兼容关系以主仓库依赖锁定文件、子模块 commit 和本页“已验证基线”为准,不要仅按包名猜测兼容性。
- 升级主仓库、SDK、固件或 XenseVR PC Service 后,应重新执行环境验证、设备自检和一条短 episode 校验。
