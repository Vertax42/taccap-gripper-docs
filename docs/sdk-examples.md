# 示例

SDK 示例脚本位于 `python/examples/`(C++ 示例需 `-DTACCAP_BUILD_EXAMPLES=ON`)。

## 快速上手

=== "单夹爪"

    ```python
    import xense.taccap as t

    # 自动发现唯一连接的夹爪(左或右);0 个或 >1 个会抛 IoError
    gripper = t.LeaderGripper.open()      # 仅 MCU;相机默认不开
    gripper.start_streaming(imu_hz=100, encoder_hz=100)

    gripper.encoder.on_data(lambda s: print("enc", s.position_rad))
    gripper.imu.on_data(lambda s: print(s))
    # ... 采集 ...
    gripper.stop_streaming()
    ```

=== "双夹爪(左右同进程)"

    ```python
    from xense.taccap import LeaderGripper, scan_grippers, Side

    endpoints = scan_grippers()           # 一次 USB 扫描拿到全部端点
    left  = next(e for e in endpoints if e.side == Side.Left)
    right = next(e for e in endpoints if e.side == Side.Right)

    g_left  = LeaderGripper(left.mcu_device)
    g_right = LeaderGripper(right.mcu_device)
    g_left.start_streaming(imu_hz=100, encoder_hz=100)
    g_right.start_streaming(imu_hz=100, encoder_hz=100)
    # ... 挂回调,退出前 stop_streaming() ...
    ```

## 示例脚本一览

| 脚本 | 作用 |
|---|---|
| `rerun_dual_with_tracker.py` | 双夹爪 IMU/编码器 + Pico4 Ultra 企业版追踪器 6-DoF 位姿,在 Rerun 单视图中可视化。需 `xensevr_pc_service_sdk` 与 XenseVR PC Service 运行 |
| `calibrate.py` | 按 SN 的编码器零点标定 CLI(见 [标定与自检](04-calibration.md)) |
| `ota_update.py` | 固件 OTA 刷写 CLI,带进度与刷后状态探测。**有风险——刷错会变砖** |
| `v4l2_probe.py` / `v4l2_sweep.py` | 手动 V4L2 拉起腕/视触觉相机(发现流程为 MCU-only,不枚举相机);SN 未烧录时也有用 |
| `leader_demo`(C++) | 单主爪 5 秒多流速率报告 |

## 编码器零点标定

```bash
python python/examples/calibrate.py TCGU01A24A0002m   # 右主爪
```

流程:解析 SN → 打印当前读数(raw 与钳位)→ 提示"保持完全闭合按 Enter" →
发送 `SetEncoderZero` → 校验残差 → 可选检查最大开合角 → 10 Hz 实时读数。

!!! tip "标定细节"
    闭合恒为 0;负向漂移会被钳到 0(原始值保留在 `raw_position_rad`);raw 负漂
    超过 -0.1 rad 会限频告警。完整说明见 [4.1 编码器零点标定](04-calibration.md)。

## Pico4 Ultra 企业版追踪器绑定(按台)

`rerun_dual_with_tracker.py` 需显式 `--left-tracker-sn` / `--right-tracker-sn`,因为
追踪器物理粘在特定夹爪上,软件无法反推。用 `scan_grippers` 报告的 SN 匹配你本机的配对:

```bash
python python/examples/rerun_dual_with_tracker.py \
    --left-tracker-sn  <左追踪器SN> \
    --right-tracker-sn <右追踪器SN>
```

!!! warning "SN 属于具体硬件"
    示例里的 SN 标识**特定机台**的设备,换机需替换为你的 `xensevr_pc_service_sdk`
    报告的追踪器 SN,并逐个摇晃夹爪验证左右对应。
