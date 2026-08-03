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
| `xense-taccap-lerobot` | 基于 lerobot 0.5.1 定制;版本号 `0.5.1+xtac.0.0.3`(与文档版本同步) | `main@395e2786` |
| `xense.taccap`(`taccap-gripper` SDK) | 与主仓库子模块版本配套 | 0.1.7(`c61b4f1`) |
| 夹爪固件协议 | 帧格式 V2.1(`hw_v1.1.0`) | leader 1.2.0 / follower 1.1.0(镜像随 SDK 附带,见 [固件 OTA 升级](#ota)) |
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

### 仓库 + 子模块

```bash
git pull --recurse-submodules
git submodule update --init --recursive --progress
./setup_env.sh --install     # 重新对齐依赖
```

!!! danger "拉完子模块必须重新编译 `xense.taccap`"
    `git submodule update` 只换源码,不重编译原生扩展。见
    [2.2 克隆仓库与子模块](02-environment.md)。

### 固件 OTA 升级 {#ota}

**什么时候需要**:固件低于 V2.1(leader 1.2.0)时没有 `Cmd::EncoderMaxCal`,
[夹爪标定](04-calibration.md#41)的第 2 步做不了,`gripper.pos` 只能走
`gripper_open_rad` 回退、够不到 1.0。除此之外不必主动升级。

SDK 自 0.1.7 起**随仓库附带已发布的固件镜像**,升级不再需要固件源码
(源码在内网仓库,不随 SDK 分发):

| 镜像 | 适用角色 | 版本 |
|---|---|---|
| `tc-gu-01-master.bin` | 主夹爪(SN 末位 **`m`**) | 1.2.0.0 |
| `tc-gu-01-slave.bin` | 从夹爪(SN 末位 **`s`**) | 1.1.0.0 |

路径 `third_party/taccap-gripper/firmware/`,同目录 `manifest.json` 记录了每个镜像的
版本、字节数、CRC32 和构建来源 commit。只保留当前发布版,历史镜像从该目录的 git 历史取。

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

!!! danger "刷错角色会变砖,需要 SWD 探针才能救回"
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
