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
    **原因**:设备 SN 从未烧录,或固件 < V1.6——是设备/固件问题,不是主机问题。
    **解决**:联系设备/固件团队烧录 SN 或升级固件。

??? failure "`ValueError` 指名某个 hub / 序列号"
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

??? failure "追踪器侧别匹配错 / PC 服务枚举不稳"
    **原因**:序列号不合规,或枚举抖动。
    **解决**:用 `--robot.tracker_serial=<SN>` 逐字钉住(不枚举、不校验);或确认序列号
    **倒数第二位**奇左偶右。

## 采集与录制

??? failure "编码器跟不上、日志出现丢帧告警"
    **原因**:实时编码队列满时会丢最旧帧(不阻塞采集循环)。
    **解决**:增大 `--dataset.encoder_threads`、用 `--dataset.vcodec=auto` 硬件编码、
    或调 `--dataset.encoder_queue_maxsize`。见 [5.5 录制选项](05-data-collection.md#55)。

??? failure "夹爪开度不对 / 闭合时不为 0"
    **原因**:编码器零点漂移或未标定。
    **解决**:重新标定零点:
    ```bash
    python third_party/taccap-gripper/python/examples/calibrate.py <固件SN>
    ```
    见 [4.1 编码器零点检查与按需标定](04-calibration.md#41)。

??? failure "腕相机/视触觉打不开、`video ... busy`"
    **原因**:相机由外部相机服务占用,或用户不在 `video` 组。
    **解决**:确认相机服务状态;把用户加入 `video` 组
    (`sudo usermod -aG video "$USER"`,重登生效)。

## 数据与磁盘

??? failure "采集变慢 / 磁盘写满"
    **原因**:双臂满负荷出流可达 **~280 MB/s**,磁盘空间/带宽不足。
    **解决**:采集前预留充足磁盘;必要时定期清理 `~/.cache/huggingface/lerobot/` 下旧数据集;
    存储规划参考待补的[数据管理]小节。

---

仍未解决?带上**完整报错**与自检输出(`scan_grippers` 的 side/role/firmware_sn),按
[版本兼容与支持] 渠道反馈(待补)。
