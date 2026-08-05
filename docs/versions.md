# 版本与支持

本手册对应的版本基线、如何查版本、如何升级,以及遇到问题怎么反馈。

## 必须升级到最新版本 {#required}

!!! danger "这不是可选项——采集前请先把四项都升到下表版本"
    整条链路是**配套**的:固件 V2.1 才有 `Cmd::EncoderMaxCal`,SDK 0.1.7 才能安全刷这版固件,
    而 `gripper.pos` 的 0–1 归一化要固件、SDK、标定三者齐了才成立。缺任何一环,
    **采集仍会"正常"跑完并落盘**,只是数据的开度刻度和别人对不上——事后从数据里看不出来。

| 组件 | 最低要求版本 | 怎么查 |
|---|---|---|
| `xense-taccap-lerobot` | `0.5.1+xtac.0.0.3` | `pip show lerobot` 或看 `pyproject.toml` |
| `xense.taccap` SDK | **0.1.7** | `python -c "import xense.taccap as t; print(t.__version__)"` |
| 夹爪固件 | **V2.1**(leader 1.2.0 / follower 1.1.0) | 见下方自检 |
| 每台 leader 的编码器标定 | 零点 + 行程上限已写入 flash | [4.1 夹爪标定](04-calibration.md#41) |

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
2. **重新编译原生扩展**,否则源码新、`.so` 旧,`import xense.taccap` 直接失败。
3. [固件 OTA 升级](#ota) —— **必须在第 2 步之后**,理由见该节。
4. [夹爪标定](04-calibration.md#41) —— 升级本身**不产生标定值**,不标就还是走回退。

!!! note "已经采过的数据不用重采"
    这里要求的是**今后采集**用统一版本。历史数据保持原样即可;若要和升级后的数据混用,
    先确认两批的 `gripper.pos` 是否落在同一刻度上(升级+标定前后不一定一致)。

## 版本兼容基线

本页将“支持范围”和“已验证基线”分开列出。实际命令与字段仍以本地 checkout 为准。

| 组件 | 支持范围 / 约束 | 已验证基线 |
|---|---|---|
| 操作系统 | Ubuntu 22.04 / 24.04 | Ubuntu 22.04.5 LTS / 24.04.4 LTS |
| Linux 内核 | 不构成约束 | 6.8 / 6.14 / 7.0 系列均已验证 |
| NVIDIA GPU / 驱动 | GPU 可选;多路视频建议使用 NVIDIA H.264 硬件编码 | 驱动 570.144 |
| Python | **≥ 3.12**(`pyproject.toml` 的 `requires-python`;`conda_environment.yaml` 固定 `python=3.12`) | 3.12.13 |
| PyTorch | `torch>=2.2.1,<2.11.0`;`torchvision>=0.21.0,<0.26.0` | 2.10.0 / torchvision 0.25.0 |
| `torchcodec` | `>=0.2.1,<0.11.0`,由 `setup_env.sh` **按当前 torch 版本自动对齐**(不匹配会强制重装) | 0.10.0 |
| PyAV | `av>=15.0.0,<16.0.0`,安装脚本固定装 **15.1.0** | 15.1.0 |
| `rerun-sdk` | `>=0.24.0,<0.27.0`(`--display_data` 用) | 0.26.2 |
| `opencv-python` | 固定 `==4.12.0.88`(XenseRobotics 各 SDK 统一) | 4.12.0.88 |
| NumPy | `>=1.26.4` | 2.2.6 |
| `xense-taccap-lerobot` | 基于 lerobot 0.5.1 定制;版本号 `0.5.1+xtac.0.0.3`(与文档版本同步) | `main@b229c19a` + PR #9(`4d4d8228`,未合并) |
| `xense.taccap`(`taccap-gripper` SDK) | 与主仓库子模块版本配套 | 0.1.7(`16412dc`) |
| 夹爪固件协议 | 帧格式 V2.1(`hw_v1.1.0`) | leader 1.2.0 / follower 1.1.0(镜像随 SDK 附带,见 [固件 OTA 升级](#ota)) |
| `xensesdk` | 由安装脚本提供 | 2.1.1 |
| XenseVR PC Service(`.deb` 守护进程) | amd64 ≥ **v0.2.0**;arm64 目前只有 v0.1.0 | v0.2.0(子模块 `6c5ff61d`)|
| `xensevr_pc_service_sdk`(pybind) | 随主仓库子模块一起编译安装 | 0.1.0 —— **包版本号没跟着服务走**,见下 |

!!! warning "`pip show xensevr-pc-service-sdk` 会显示 0.1.0,这是正常的"
    Python 侧的 `xensevr_pc_service_sdk` 是主仓库里的 pybind 封装,`setup.py` 里的版本号
    一直是 `0.1.0`,**没有随 PC Service 的 v0.2.0 提升**。所以**不要用它判断有没有头显相机支持**——
    要看的是有没有相机接口:

    ```bash
    python -c "import xensevr_pc_service_sdk as xrt; print(hasattr(xrt, 'has_pico_camera_frame'))"
    ```

    服务本体的版本用 `dpkg -s xensevr-pc-service` 查(见[如何查版本](#check-versions))。

!!! note "PC Service v0.2.0 只影响头显相机"
    v0.2.0 相对 v0.1.0 的行为差异只有一条:**转发 `0x30`(带时间戳视频帧)消息**,
    这是[头显相机](05-data-collection.md#57)画面的通路。`0x30` 在 v0.1.0 里没有使用,
    老版本头显 APK 对 v0.2.0 也照常工作,所以**不用头显相机就没有任何行为变化**。

    v0.2.0 只发布了 amd64 包:`setup_env.sh` 在 arm64 上会自动退回 v0.1.0 并打印说明
    (代价是 arm64 没有头显相机),需要时用仓库的 `RoboticsService/qt-gcc_aarch64.sh` 自行编译。

!!! note "版本以本地为准"
    命令与字段应以你 checkout 的 `src/lerobot/robots/taccap_gripper/README.md` 与
    `third_party/taccap-gripper/` 为准。

!!! warning "头显相机相关内容对应尚未合并的 PR #9"
    本手册中的[头显相机](05-data-collection.md#57)、Insight 链路移除、以及
    [4.4](04-calibration.md#44) 里 Rerun 视图的改动,对应主仓库
    [PR #9](https://github.com/Vertax42/xense-taccap-lerobot/pull/9)(`XenseVR-PC-Service_0.2.0_merge`),
    **该 PR 目前尚未合入 `main`**。

    如果你的 checkout 停在 `main@b229c19a`:`--robot.enable_head_camera` 在单夹爪上还不存在,
    双夹爪上它指的是旧的 Insight 相机,Rerun 里也仍然画着 TRACKER 坐标系和虚线。
    合并后 `git pull --recurse-submodules` + `./setup_env.sh --install` 即与本手册一致。

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

# XenseVR PC Service 守护进程的 deb 版本(服务本体的版本以这个为准)
dpkg -s xensevr-pc-service 2>/dev/null | grep -E '^(Package|Version|Architecture):'
# pybind 层是否带头显相机接口(需要 PC Service v0.2.0;包版本号不会变,只能看接口)
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
    `git submodule update` 只换源码,不重编译原生扩展。见
    [2.2 克隆仓库与子模块](02-environment.md)。

### 固件 OTA 升级 {#ota}

**所有夹爪都要升到 V2.1**(见 [必须升级到最新版本](#required))。低于 V2.1 的固件没有
`Cmd::EncoderMaxCal`,[夹爪标定](04-calibration.md#41)的第 2 步做不了,`gripper.pos`
只能走回退、够不到 1.0。

SDK 自 0.1.7 起**随仓库附带已发布的固件镜像**,直接刷即可:

| 镜像 | 适用角色 | 版本 |
|---|---|---|
| `tc-gu-01-master.bin` | 主夹爪(SN 末位 **`m`**) | 1.2.0.0 |
| `tc-gu-01-slave.bin` | 从夹爪(SN 末位 **`s`**) | 1.1.0.0 |

路径 `third_party/taccap-gripper/firmware/`,同目录 `manifest.json` 记录了每个镜像的
版本、字节数与 CRC32,可用于核对文件是否完整。该目录只保留当前发布版。

!!! warning "顺序:**先升级 SDK,再刷固件**"
    0.1.7 之前的 `OtaSession` 只检查 `ack.is_nack`,而固件侧的错误是从**回显命令**这条路
    返回的——传输层无法与"1 字节的成功应答"区分。结果是写入被固件拒绝却不报错,
    **失败的升级会报成功**;它还会用新的序列号重试,而固件把这当成一次新请求并因偏移不连续
    整段拒绝,一次仅仅是慢了的 ACK 就能毁掉一次本来正常的升级。两者都在 0.1.7 修好了。

    新 SDK 与旧固件通信不变(V1.9 以前的命令集没动),所以**先升 SDK 总是安全的**。

**按角色选镜像,不是按左右手。**角色看固件 SN 的**最后一个字符**:
`TCGU01A28Z0023m` → `m` → master。同一套双臂设备上两只夹爪常常**都是 master**。

```bash
# 1. 确认每只夹爪的角色
python -c "from xense.taccap import scan_grippers
for g in scan_grippers(): print(g.firmware_sn, '->', 'master' if g.firmware_sn.endswith('m') else 'slave')"

# 2. 刷写(镜像只写文件名即可,脚本会去 SDK 的 firmware/ 里找)
python third_party/taccap-gripper/python/examples/ota_update.py \
    tc-gu-01-master.bin --side left --target-version 1.2.0.0

# 3. 确认:GetVersion 返回固件编译进去的常量,读回的版本就是实际刷上去的版本
python -c "from xense.taccap import scan_grippers
for g in scan_grippers(): print(g.firmware_sn, g.role.name)"
```

约 1 秒写完,MCU 重启并重新枚举 USB 约 1–3 秒。写入走的是**非活动 flash bank**,
`OtaApply` 之前不覆盖任何东西,CRC 不匹配时固件拒绝切换——传输失败不会损坏正在运行的固件。

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
升级本身不产生标定值,`gripper.pos` 仍会走回退直到标定完成。

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
- 主仓库版本号与本页文档版本对齐:`xense-taccap-lerobot` 的 `pyproject.toml` 记 `0.5.1+xtac.0.0.3`,其中 `0.5.1` 是上游 lerobot 基线,`xtac.0.0.3` 是与本文档同步的产品版本。
- 精确兼容关系以主仓库依赖锁定文件、子模块 commit 和本页“已验证基线”为准,不要仅按包名猜测兼容性。
- 升级主仓库、SDK、固件或 XenseVR PC Service 后,应重新执行环境验证、设备自检和一条短 episode 校验。
