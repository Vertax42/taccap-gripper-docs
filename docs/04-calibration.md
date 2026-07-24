# 4. 标定与自检

对应参考手册的"测试网络连接"并扩展。正式录制前先检查夹爪完全闭合时的编码器读数,
确认整条链路可用。读数正常时无需重复标定。

## 4.1 编码器零点检查与按需标定

**零点标定不需要每次执行。**先将夹爪完全闭合并检查读数;只有完全闭合时读数仍不为 0,
或出现明确的零点漂移 / 告警时,才使用 SDK 标定 CLI 调用 `Encoder.set_zero()` 重新锁定零点、校验
零点后残差,并可选检查开合角。

```bash
python third_party/taccap-gripper/python/examples/calibrate.py SN000003
```

先列出可用的固件 SN:

```bash
python -c "from xense.taccap import scan_grippers, Side; \
  [print(f'{\"L\" if g.side==Side.Left else \"R\"} fw={g.firmware_sn} mcu={g.mcu_serial}') for g in scan_grippers()]"
```

标定 CLI 的流程:

1. 把固件 SN 解析到对应的 `mcu_device`。
2. 打印当前编码器读数(`raw` 与钳位后)以便看到现有漂移。
3. 提示"**保持夹爪完全闭合**,按 [Enter]"。
4. 发送 `Cmd::SetEncoderZero`,重读,校验新 `raw` 在容差内(默认 ±0.01 rad)。
5. 可选 `Step 2/2`:探测机械最大开合角,与期望包络比较(默认 1.7 rad ≈ 97°,
   `--expected-max-open-rad` 可调)。
6. 10 Hz 实时读数(`raw | cooked`)直到 Ctrl+C。

!!! tip "闭合 = 0,是约定"
    零点锁定后,`position_rad` 闭合时读 0,机械极限时升到 ~1.7 rad。**没有
    `gripper_closed_rad` 配置**——闭合永远是 0;每台只可配 `gripper_open_rad`(默认 1.7)。

!!! warning "必须先摆好姿态再按 Enter"
    固件在处理命令的瞬间锁存当时看到的原始计数,所以按 Enter 前夹爪必须已在目标(闭合)姿态。

## 4.2 Pico4 Ultra 企业版追踪器自检

```bash
python -m lerobot.robots.taccap_gripper.calibrate_tracker
# 或指定某个 tracker SN:
python -m lerobot.robots.taccap_gripper.calibrate_tracker LHR-XXXXXXXX
```

挥动夹爪时观察 `raw xyz` 是否平滑变化。`ee xyz` 是 `raw` 经过刚性
`tracker_to_ee_*` 安装变换后的结果(默认单位变换)。测量你的物理安装偏移并写入配置
(`tracker_to_ee_pos`、`tracker_to_ee_quat`)。

## 4.3 独立冒烟测试

在**不依赖 `lerobot-record`** 的情况下验证整个机器人栈。设备自动发现;仅当两只夹爪都
接入时才需 `--side`。

=== "夹爪 + 触觉 + 腕相机(全部自动发现)"

    ```bash
    python -m lerobot.robots.taccap_gripper.taccap_gripper_example --side left
    ```

=== "仅相机 + 夹爪(无腕相机)"

    ```bash
    python -m lerobot.robots.taccap_gripper.taccap_gripper_example --side left --no-wrist
    ```

=== "+ Pico4 Ultra 企业版追踪器(位姿)"

    ```bash
    python -m lerobot.robots.taccap_gripper.taccap_gripper_example \
        --side left --tracker --tracker-sn PT-XXXXXXXXXXXX
    ```

## 4.4 3D 轨迹可视化(Rerun) {#44}

录制/自检时加 `--display_data=true`,Rerun 查看器会多出一个 `/world` 3D 视图:夹爪
以带标签的椭球 + 坐标三轴在其实时 Pico4 Ultra 企业版位姿(`tcp.*`)处绘制,并拖出一条走过的
轨迹面包屑。

- 我们的位姿已在重力对齐世界系,场景是 `RIGHT_HAND_Z_UP`。
- 默认开启;`--show_trajectory=false` 关闭;当 `--robot.enable_tracker=false`
  (无位姿可画)时自动跳过。

!!! note "参考实现"
    与 SDK 的 `python/examples/rerun_dual_with_tracker.py` 示例一致(该示例展示的是
    Pico4 Ultra 企业版原始 `LEFT_HAND_Y_UP` 系)。

标定与自检通过后,即可开始正式采集。

下一步 → [5. 数据采集](05-data-collection.md)
