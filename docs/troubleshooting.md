# 故障排查

按**症状**分类。每条:症状 → 原因 → 解决。多数问题集中在**串口权限**与 **ModemManager 抢占**——先看这两类。

!!! tip "定位思路"
    先跑自检 `python -c "from xense.taccap import scan_grippers; ..."`(见 [一页速通 §2](quickstart.md)),
    再对照下面的症状。

## 环境与安装

??? failure "`import xensesdk` / `import xensevr_pc_service_sdk` / `import xense.taccap` 失败"
    **原因**:环境未装全,或未激活 `xense-taccap` 环境。
    **解决**:
    ```bash
    mamba activate xense-taccap
    ./setup_env.sh --install     # 重跑安装
    ```
    逐个验证见 [环境部署 §2.5](02-environment.md#25)。

??? failure "`torchcodec` 加载失败 / 视频编码报错"
    **原因**:`torchcodec` 与当前 PyTorch 的兼容版本不匹配,或 PyAV 不是要求的 `15.1.0`。
    **解决**:重跑 `setup_env.sh --install` 自动校正版本;需要带 `libsvtav1` 的系统 FFmpeg 时再单独安装。

## 串口权限与设备发现

??? failure "`connect()` 报 `No leader gripper discovered for the <side> side.`"
    **原因**(最常见):用户不在 `dialout` 组,SDK 能*列出*夹爪但打不开串口读固件 SN,
    于是 `role=Unknown` / `firmware_sn` 为空。底层是
    `IoError: SerialBus: open(...): Permission denied`。
    **解决**:
    ```bash
    sudo usermod -aG dialout "$USER"
    # 注销重登(或 newgrp dialout),然后重插夹爪
    ```
    详见 [3.1 串口权限](03-host-hardware.md#31)。

??? failure "`Device or resource busy`(热插拔后立即启动)"
    **原因**:**ModemManager** 每次热插拔都用 AT 指令探测 CH343 串口并占用几秒。典型:
    第一次启动正常,拔下→换口→立即重启就 busy。(装了 `brltty` 也会同样抢占。)
    **解决**:临时——插好等 ~3 秒;永久——udev 规则忽略 `1a86` 设备:
    ```bash
    sudo tee /etc/udev/rules.d/99-taccap-ignore-modemmanager.rules >/dev/null <<'EOF'
    ACTION=="add|change", SUBSYSTEMS=="usb", ATTRS{idVendor}=="1a86", ENV{ID_MM_DEVICE_IGNORE}="1"
    EOF
    sudo udevadm control --reload-rules && sudo udevadm trigger
    ```
    详见 [3.2 关闭 ModemManager 抢占](03-host-hardware.md#32)。

??? failure "`firmware_sn` 修好权限后仍为空 / `role=Unknown`"
    **原因**:可能是设备 SN 未烧录、串口读取仍失败、固件通信异常或设备端配置问题;不能只根据空 SN 推断某个固件版本。
    **解决**:先保存完整底层报错并换线 / 换口复测;仍为空时联系设备或固件团队核对 SN 烧录与通信状态。

??? failure "报错指名某个 hub / 序列号"
    **原因**:发现阶段检测到硬件装配与"单左双右"规则不符——序列号不合规、每侧数量不对、
    两传感器映射到同一手指、触觉 hub 找不到对应夹爪。
    **解决**:按报错**指名的物理设备/hub**排查装配与接线。规则见 [3.3 设备发现](03-host-hardware.md#33)。

## Pico4 Ultra 企业版追踪器与位姿

??? failure "没有位姿 / 追踪器连不上 / 位姿不稳"
    **原因**:**电脑 WiFi 与 Pico4 Ultra 企业版有线共享网络冲突**(最常见);或 XenseVR PC Service 未启动、
    XenseVR-Toolkit 未启动、追踪器未配对/没电。
    **解决**:**先关闭数采电脑 WiFi**(只保留 Pico4 Ultra 企业版有线共享网络,见
    [3.4 网络连接](03-host-hardware.md#pico-network));再按 [上电顺序](03-host-hardware.md#36)
    逐项确认;启动服务 `/opt/apps/roboticsservice/runService.sh`;必要时用
    `python -m lerobot.robots.taccap_gripper.calibrate_tracker` 自检。

??? failure "位姿参考系在集之间漂移"
    **原因**:分集途中**重启了 XenseVR-Toolkit**,世界原点被重设。
    **解决**:采集**全程不要重启** XenseVR-Toolkit。见 [3.4 Pico4 Ultra 企业版配置](03-host-hardware.md#34)。

??? failure "采集间隙位姿突然中断 / 头显自己灭屏"
    **原因**:头显未关闭灭屏与系统休眠,挂起后 XenseVR-Toolkit 被系统暂停或杀掉;
    重启 Toolkit 又会重新冻结世界系,叠加出上一条的参考系漂移。
    **解决**:企业设置 → 系统设置 → **电源策略**,**先**把「系统休眠」设为「永不」,
    **再**把「灭屏」设为「永不」(顺序反了灭屏会被休眠时间钳制回有限值)。
    见 [首次安装 · 步骤 2](03-host-hardware.md#pico-app)。

??? failure "追踪模式里选不到追踪器 / PC Service 发现不了这个 SN"
    **原因**:追踪器**没有绑定到这台头显**(新机、换了追踪器、换了头显或恢复过出厂设置)。
    **解决**:打开「体感追踪器」App 完成绑定,两枚都要绑。
    见 [绑定运动追踪器到头显](03-host-hardware.md#pico-tracker-bind)。

??? failure "配对界面搜不到追踪器"
    **原因**:追踪器只是普通开机(**蓝灯常亮**),没进入蓝牙配对状态。
    **解决**:**长按电源键约 6 秒**,直到指示灯**蓝红交替闪烁**,再点「开始配对」。
    见 [绑定运动追踪器到头显](03-host-hardware.md#pico-tracker-bind)。

??? failure "头显里显示已连上,PC 端却收不到任何位姿"
    **原因**:APP 里 **`Send` 没勾**(数据推送总开关);或 tracker 下拉未选 `Object`。
    **解决**:按 [界面清单](03-host-hardware.md#pico-toolkit-ui) 逐项核对:
    连网络 → tracker = `Object` → 勾 `High-Acc` → 勾 `Send`。

??? failure "改了 Mode 或 High-Acc 但 PC 端行为没变"
    **原因**:`Send` 已经勾着,设置是在数据流开着的时候改的。
    **解决**:**取消 `Send` 再重新勾选**,让数据流带新设置重开(不要重启 APP,重启会重置世界系)。
    见 [界面清单](03-host-hardware.md#pico-toolkit-ui)。

??? failure "追踪器侧别匹配错 / PC 服务枚举不稳"
    **原因**:序列号不合规,或枚举抖动。
    **解决**:用 `--robot.tracker_serial=<SN>` 逐字钉住(不枚举、不校验);或确认序列号
    **末尾字母 `G` 前一个数字**:单左双右。

## 头显相机 {#head-camera}

??? failure "开了 `--robot.enable_head_camera=true`,一直卡在等待首帧"
    **原因**:相机画面由 PC Service 转发,这条路上任何一环没通都收不到帧:
    服务版本低于 v0.2.0(v0.1.0 不转发头显相机画面)、头显 APP 没在推流。
    **解决**:按这个顺序查——

    ```bash
    # 1) 服务 deb 版本:amd64 需要 0.2.0
    dpkg -s xensevr-pc-service | grep -E '^(Version|Architecture):'
    # 2) 是否带相机接口
    python -c "import xensevr_pc_service_sdk as xrt; print(hasattr(xrt, 'has_pico_camera_frame'))"
    ```

    再确认头显已连上 PC Service 且 APP 正在推流(相机与追踪器**共用同一条连接**)。
    见 [5.7 头显相机 · 前置条件](05-data-collection.md#57)。

??? failure "`AttributeError: module 'xensevr_pc_service_sdk' has no attribute 'has_pico_camera_frame'`"
    **原因**:环境里加载的是**旧版接口**(相机接口是随 v0.2.0 一起加的)。
    **解决**:拉最新主仓库与子模块后重跑 `./setup_env.sh --install`;仍然为 `False` 时
    用 `python -c "import xensevr_pc_service_sdk as x; print(x.__file__)"` 确认加载的是哪一份。

??? failure "arm64 主机装不上 v0.2.0 / 下载 404"
    **原因**:v0.2.0 **只发布了 amd64 包**。`setup_env.sh` 检测到 arm64 会自动退回 v0.1.0
    并打印说明——所以 arm64 上**没有头显相机**,这是预期行为,不是安装失败。
    **解决**:确实需要时,在 ARM64 主机上用 `RoboticsService/qt-gcc_aarch64.sh` 自行编译 v0.2.0。

??? failure "日志反复出现左右眼偏差(skew)告警"
    **原因**:两只眼是两条独立消息,帧序号不同且时间戳差超过
    `--robot.head_camera_pair_max_skew_ms`(默认 20 ms)。常见于主机负载过高、
    或头显与 PC 之间的链路抖动。
    **解决**:告警**不会中断录制**,但这几帧的左右眼可能不是同一次曝光。先降低主机负载
    (减少相机路数、用 `--robot.head_camera_eyes=left` 只录一只眼);链路问题按
    [3.4 网络连接](03-host-hardware.md#pico-network)排查。确认只是抖动时,再考虑适当放宽该阈值。

??? failure "`head_camera_width/_height` 报错说尺寸不支持"
    **原因**:头显相机**只接受 `1024x768` 和 `1280x960`**(都是 4:3,与传感器一致)。
    填别的会直接报错而不是悄悄降级——重采样会无声改掉记录下来的视场角。
    **解决**:改回两个受支持的尺寸之一。注意**改尺寸等于换了一组数据**,变更前后的 episode
    不能混用。见 [5.7 头显相机](05-data-collection.md#57)。

## 采集与录制

??? failure "开了 `--display_data=true` 后日志频繁出现 `[slow_frame]`"
    **原因**:Rerun 显示本身占掉了帧预算。实测双夹爪 + 头显(四路触觉、两路腕相机、两只眼),
    JPEG 压缩后每帧 13.2 ms,不压缩 3.1 ms——30 fps 的预算只有 33.3 ms,压缩就占了 40%,
    这还没算相机读取和位姿计算。

    **先看 `[slow_frame]` 那行末尾的 `top_obs=`**:是某个传感器慢,还是显示贵,两者在超时
    数字上看着一样,解法却相反。

    **解决**:两个默认值本来就是快的那个(`--display_compressed_images=false`、
    `--display_image_every_n=1`),先确认没被改过。仍然超时再把 `--display_image_every_n`
    调大——相机画面刷新变稀,`tcp.*` 和 `gripper.pos` 等标量仍是全速。这是最后手段,
    因为它是唯一会改变操作员所见内容的选项。

??? failure "编码器跟不上、日志出现丢帧告警"
    **原因**:实时编码队列满时会丢最旧帧(不阻塞采集循环)。
    **解决**:增大 `--dataset.encoder_threads`、用 `--dataset.vcodec=auto` 硬件编码、
    或调 `--dataset.encoder_queue_maxsize`。见 [5.5 录制选项](05-data-collection.md#55)。

??? failure "夹爪开度不对 / 闭合时不为 0"
    **原因**:编码器零点漂移或未标定。
    **解决**:重新标定零点:
    ```bash
    python third_party/taccap-gripper/python/examples/calibrate.py left    # 或 right
    ```
    该命令同时重标零点与行程上限,两者都会写入 MCU flash。见 [4.1 夹爪标定](04-calibration.md#41)。

??? failure "标定报 `encoder-max calibration needs command set >= V2.1`"
    **原因**:这台夹爪的固件命令集低于 V2.1(即 leader < 1.2.0),不支持行程标定。
    `calibrate.py` 会**原样退出,不改动任何东西**——不会留下"标了零点但没标行程"的半成品。
    **解决**:刷固件。SDK 自 0.1.7 起把已发布镜像放在
    `third_party/taccap-gripper/firmware/`,不再需要固件源码:
    ```bash
    python third_party/taccap-gripper/python/examples/ota_update.py \
        tc-gu-01-master.bin --side left
    ```
    **先升 SDK 再刷固件**(顺序反了会踩到"失败却报成功"的旧 bug),镜像**按角色选**——
    看固件 SN 末位 `m`/`s`,不是看左右手。完整步骤与风险见
    [固件 OTA 升级](versions.md#ota)。刷完回来重跑 `calibrate.py`。

??? failure "旧版本提示到 `third_party/firmware/…` 下找 .bin,但该目录不存在"
    **原因**:旧版本脚本里的一处过时路径,指向的目录不随 SDK 分发。
    **解决**:改用随 SDK 附带的镜像,镜像名直接写 `tc-gu-01-master.bin` /
    `tc-gu-01-slave.bin` 即可(脚本会去 SDK 的 `firmware/` 里找),见
    [固件 OTA 升级](versions.md#ota)。SDK 0.1.7 之后已修正,按本页升级后不会再出现。

??? failure "腕相机/视触觉打不开、`video ... busy`"
    **原因**:相机由外部相机服务占用,或用户不在 `video` 组。
    **解决**:确认相机服务状态;把用户加入 `video` 组
    (`sudo usermod -aG video "$USER"`,重登生效)。

## 数据与磁盘

??? failure "采集变慢 / 磁盘写满"
    **原因**:双夹爪多相机的原始视频数据吞吐可达约 280 MB/s;实时编码后的实际磁盘写入量取决于分辨率、画面内容、编码器与码率。目标盘空间不足、持续写入性能不足或编码线程跟不上都会导致异常。
    **解决**:先用少量 episode 实测编码后体积和丢帧情况,再规划批量采集;定期检查 `df -h` 与数据集目录大小。详见[数据管理与命名](data-management.md#storage-planning)。

---

仍未解决?请带上完整报错、自检输出(`scan_grippers` 的 side / role / firmware_sn)、版本信息和复现命令,按[版本与支持](versions.md#support)中的渠道反馈。
