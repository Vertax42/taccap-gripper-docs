# 示例

SDK 示例脚本位于 `python/examples/`(C++ 示例需 `-DTACCAP_BUILD_EXAMPLES=ON`)。

## 快速上手

=== "Python:单主夹爪"

    ```python
    import xense.taccap as t

    # 按固件 SN 的 m 后缀选择 Leader,避免把 Follower 交给 LeaderGripper。
    ep = t.find_leader()
    gripper = t.LeaderGripper(ep.mcu_device)  # MCU-only;腕相机默认不开

    gripper.encoder.on_data(lambda s: print("enc", s.position_rad))
    gripper.imu.on_data(lambda s: print(s))
    gripper.start_streaming(imu_hz=100, encoder_hz=100)
    try:
        # ... 采集 ...
        pass
    finally:
        gripper.stop_streaming()
    ```

=== "Python:双主夹爪"

    ```python
    from xense.taccap import LeaderGripper, Role, Side, scan_grippers

    endpoints = scan_grippers()
    left = next(e for e in endpoints
                if e.side == Side.Left and e.role == Role.Leader)
    right = next(e for e in endpoints
                 if e.side == Side.Right and e.role == Role.Leader)

    grippers = [LeaderGripper(left.mcu_device), LeaderGripper(right.mcu_device)]
    try:
        for g in grippers:
            g.start_streaming(imu_hz=100, encoder_hz=100)
        # ... 为两只夹爪挂回调并采集 ...
    finally:
        for g in grippers:
            g.stop_streaming()
    ```

=== "C++17:单主夹爪"

    ```cpp
    #include <taccap/discovery.hpp>
    #include <taccap/leader_gripper.hpp>

    #include <chrono>
    #include <iostream>
    #include <thread>

    int main() {
        namespace tc = xense::taccap;

        const auto ep = tc::discovery::find_leader();
        tc::LeaderGripper::Config cfg;
        cfg.mcu_device = ep.mcu_device;
        tc::LeaderGripper gripper(cfg);

        gripper.encoder().on_data([](const auto& sample) {
            std::cout << "encoder=" << sample.position_rad << "\n";
        });
        gripper.start_streaming(100, 100);
        std::this_thread::sleep_for(std::chrono::seconds(5));
        gripper.stop_streaming();
        return 0;
    }
    ```

    工程中通过 `add_subdirectory()` 引入 SDK 并链接 `taccap_core`,详见[安装与构建](sdk-install.md)。

## 示例脚本一览

| 脚本 | 作用 |
|---|---|
| `rerun_dual_with_tracker.py` | 双夹爪 IMU/编码器 + Pico4 Ultra 企业版追踪器 6-DoF 位姿,在 Rerun 单视图中可视化。需 `xensevr_pc_service_sdk` 与 XenseVR PC Service 运行 |
| `calibrate.py` | 按 SN 检查编码器零点并按需重新标定(见 [标定与自检](04-calibration.md)) |
| `ota_update.py` | 固件 OTA 刷写 CLI,带进度与刷后状态探测。**有风险——刷错会变砖** |
| `v4l2_probe.py` / `v4l2_sweep.py` | 直接用 SDK `Camera` 调试 V4L2/UVC 节点;仅用于底层排障,不代表正式 LeRobot / `xensesdk` 图像采集路径 |
| `motor_mit_control.py` | 从夹爪原始弧度坐标下的 MIT 阻抗控制演示;会驱动真实电机 |
| `gripper_control_test.py` | 从夹爪归一化开度 `[0,1]` 与 `ControlLoop` 交互测试;要求从夹爪配置已标定 |
| `leader_demo`(C++) | 单主夹爪 IMU + 编码器 5 秒 MCU 流速率报告 |

## C++ 冒烟程序

启用 C++ 示例后,可直接运行官方 `leader_demo` 验证单只主夹爪的 IMU 与编码器流:

```bash
cmake -B build -G Ninja \
    -DTACCAP_BUILD_PYTHON=OFF \
    -DTACCAP_BUILD_EXAMPLES=ON
cmake --build build -j
./build/cpp/examples/leader_demo
```

该程序自动发现当前唯一连接的夹爪并采样 5 秒。若同时连接多只夹爪,应改用显式端点构造,不要使用 `LeaderGripper::open()`。

## 从夹爪电机控制(高风险)

!!! danger "会驱动真实电机"
    以下脚本仅适用于从夹爪(Follower)。运行前清空夹爪运动范围,确认急停 / 断电手段可用,并保持手指远离夹爪。主夹爪没有电机,不要在主夹爪上执行。

- `motor_mit_control.py` 直接发送原始电机弧度目标和 MIT 阻抗参数,适合底层控制链路调试。
- `gripper_control_test.py` 使用归一化开度 `[0,1]` 测试单次命令与 `ControlLoop`;要求从夹爪的 `GripperConfig` 已完成闭合零点和最大开度标定。

```bash
python python/examples/motor_mit_control.py --hz 200 --seconds 5
python python/examples/gripper_control_test.py
```

## 夹爪标定(零点 + 行程)

`calibrate.py` **每台主夹爪需执行一次**:标定编码器零点,并把该夹爪的行程上限写入 MCU flash。
后者是 `gripper.pos` 归一化到 `1.0` 的依据,缺了它软件会退回除以配置常量。值存在 flash 中,
之后无需重复运行。

```bash
python python/examples/calibrate.py TCGU01A24A0002m   # 右主爪
```

流程:解析 SN → 打印当前读数(raw 与钳位)→ 提示"保持完全闭合按 Enter" →
发送 `SetEncoderZero` → 校验残差 → 提示"张开到机械极限按 Enter" →
采样并写入 `EncoderMaxCal` → 10 Hz 实时读数。

!!! tip "标定细节"
    闭合恒为 0;负向漂移会被钳到 0(原始值保留在 `raw_position_rad`);raw 负漂
    超过 -0.1 rad 会限频告警。完整说明见 [4.1 夹爪标定](04-calibration.md#41)。

## Pico4 Ultra 企业版追踪器绑定(按台)

作为 SDK 独立示例,`rerun_dual_with_tracker.py` 不使用 LeRobot 的序列号自动匹配逻辑,
因此需显式传入 `--left-tracker-sn` / `--right-tracker-sn`。正式 `lerobot-teleoperate` /
`lerobot-record` 流程默认可按追踪器序列号规则自动匹配。

```bash
python python/examples/rerun_dual_with_tracker.py \
    --left-tracker-sn  <左追踪器SN> \
    --right-tracker-sn <右追踪器SN>
```

!!! warning "SN 属于具体硬件"
    示例里的 SN 标识**特定机台**的设备,换机需替换为你的 `xensevr_pc_service_sdk`
    报告的追踪器 SN,并逐个摇晃夹爪验证左右对应。
    读法见 [读取追踪器 SN](03-host-hardware.md#pico-tracker-sn)。
