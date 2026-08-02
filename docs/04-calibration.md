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
# 指定某个 tracker SN(格式见 3.3,形如 PC2310MLL3200496G):
python -m lerobot.robots.taccap_gripper.calibrate_tracker <tracker SN>
# 应用该侧内置的 tracker→TCP 安装变换:
python -m lerobot.robots.taccap_gripper.calibrate_tracker --side right
```

以 10 Hz 打印 `raw`(追踪器自身位姿)与 `ee`(经刚性安装变换后的 TCP)。挥动夹爪,
`raw xyz` 应平滑变化、SN 与预期一致。

!!! note "安装变换是内置的,不需要你测"
    追踪器拧在夹爪上,它报的是**追踪器**的位姿,不是我们要记的 TCP。两者之间的刚性偏移
    由 `ee_transform.tracker_to_tcp` 内置(取自 CAD 装配实测),**左右两侧各自实测**——
    两侧接近镜像但不完全相同(旋转差 0.03°、平移差 1.27 mm),所以左值不是把右值镜像出来的。

    `--side` 决定套用哪一侧的内置值;**不带 `--side` 时变换是单位阵**,`ee` 会完全跟随 `raw`。
    需要覆盖(如重新加工过的安装座)才设 `--robot.tracker_to_ee_pos` /
    `--robot.tracker_to_ee_quat`,两者独立,可以只钉平移、旋转仍用内置值。

!!! tip "支点检查:不用任何额外硬件就能验证安装变换"
    把夹爪**两指的中点**抵在一个固定点上,握着手柄尽可能多地变换姿态摆动。
    **`ee xyz` 应基本不动,而 `raw xyz` 大幅摆动**——这就是全部测试内容,
    看到的漂移量即该变换的误差。**左右两侧都要测**;若左侧的值镜像方向错了,
    表现为 `ee` 的摆动幅度约为应有的两倍。

四元数出现半球翻转(符号跳变)时,reader 内有连续性修正;**若仍看到跳变请报 bug**。

## 4.3 端到端冒烟测试

使用当前实际支持的 `lerobot-teleoperate` 入口验证夹爪、触觉和腕相机数据流。该命令只读取并预览设备,
**不会录制数据集**。为单独检查夹爪本体与相机,下面先关闭 Pico4 追踪器,运行 10 秒后自动退出。

**单夹爪(以右侧为例):**

```bash
lerobot-teleoperate \
    --robot.type=taccap_gripper \
    --robot.side=right \
    --robot.enable_tracker=false \
    --fps=30 \
    --teleop_time_s=10 \
    --debug_timing=true
```

**双夹爪:**

```bash
lerobot-teleoperate \
    --robot.type=bi_taccap_gripper \
    --robot.enable_tracker=false \
    --fps=30 \
    --teleop_time_s=10 \
    --debug_timing=true
```

命令能持续输出采样耗时与相机数量,且无设备发现、连接或读取异常,即表示基础数据流正常。

## 4.4 3D 轨迹可视化(Rerun) {#44}

使用 `lerobot-teleoperate` 并开启 `--display_data=true`,即可实时预览全部观测和 3D 轨迹。

**单夹爪(以右侧为例):**

```bash
lerobot-teleoperate \
    --robot.type=taccap_gripper \
    --robot.side=right \
    --fps=30 \
    --display_data=true \
    --show_trajectory=true
```

**双夹爪:**

```bash
lerobot-teleoperate \
    --robot.type=bi_taccap_gripper \
    --fps=30 \
    --display_data=true \
    --show_trajectory=true
```

Rerun 查看器会多出一个 `/world` 3D 视图:夹爪以带标签的椭球 + 坐标三轴在其实时
Pico4 Ultra 企业版位姿(`tcp.*`)处绘制,并拖出一条走过的轨迹面包屑。

- 我们的位姿已在重力对齐世界系,场景是 `RIGHT_HAND_Z_UP`,世界轴带
  `+X forward / +Y left / +Z up` 标注。
- 机器人同时发布追踪器自身位姿时,Rerun 会**在 EE 坐标系旁再画一个更小更暗的追踪器坐标系**
  (仅显示,不落盘),两者之间连一条虚线并标出**长度(mm)**。这条长度是**判断安装变换对不对
  最快的一眼**:采集全程它应保持恒定;数值跳动或明显偏离机械尺寸,说明变换或装配有问题。
- `--show_trajectory` 默认开启;设为 `false` 可关闭。当 `--robot.enable_tracker=false`
  (无位姿可画)时会自动跳过。

!!! note "与 SDK 独立示例的区别"
    轨迹标记与面包屑的可视化形式和 SDK `python/examples/rerun_dual_with_tracker.py` 相似;正式 LeRobot 流程使用重力对齐的 `RIGHT_HAND_Z_UP` 世界系,SDK 独立示例展示 Pico4 原始 `LEFT_HAND_Y_UP` 坐标系。

标定与自检通过后,即可开始正式采集。

下一步 → [5. 数据采集](05-data-collection.md)
