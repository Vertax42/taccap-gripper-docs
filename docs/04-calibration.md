# 4. 标定与自检

对应参考手册的"测试网络连接"并扩展。正式录制前先完成夹爪标定,再逐项确认整条链路可用。

## 4.1 夹爪标定(零点 + 行程) {#41}

### 4.1.1 为什么必须标

数据集里的 `gripper.pos` 是**归一化开度**:`0.0` 完全闭合,`1.0` 完全张开。这两个端点不是
算出来的,是标定写进 MCU flash 的两个数:

| 端点 | 来源 | 命令 |
| --- | --- | --- |
| `0.0` 闭合 | 编码器零点 | `Cmd::SetEncoderZero` |
| `1.0` 张开 | 该夹爪的行程上限(encoder max) | `Cmd::EncoderMaxCal`(固件 ≥ V2.1) |

**没标行程上限会怎样。**软件回退成除以配置常量 `gripper_open_rad`(默认 `1.7`)——一个数
代表所有出厂夹爪。但每台的真实行程不同:实测某台是 **1.1486 rad(65.8°)**,在回退算法下
张到底只能读到 `1.1486 / 1.7 = 0.676`,**永远够不到 1.0**,而且这个上限每台各异。
策略学到的"完全张开"因此和物理动作对不上。

!!! danger "双臂只标一侧,比两侧都不标更糟"
    两侧都没标时刻度至少一致;只标一侧会让 `left_gripper.pos` 和 `right_gripper.pos`
    落在**不同刻度**上——同一个握持动作左右读数不同,而数据里看不出任何异常。
    **要标就两侧都标。**

### 4.1.2 怎么标

先列出在线夹爪的固件 SN:

```bash
python -c "from xense.taccap import scan_grippers, Side; \
  [print(f'{\"L\" if g.side==Side.Left else \"R\"} fw={g.firmware_sn} mcu={g.mcu_serial}') for g in scan_grippers()]"
```

对**每一台** leader 夹爪执行(SN 换成上一步列出的):

```bash
python third_party/taccap-gripper/python/examples/calibrate.py TCGU01A28Z0024m
```

一条命令走完两步,按提示操作:

1. **保持夹爪完全闭合** → 回车。锁存为编码器零点,随后复读校验残差(容差 ±0.01 rad)。
2. **张开到机械极限** → 回车。采样该姿态的角度,确认后写入 MCU flash 作为 encoder max。

输出形如:

```text
Step 1 — hold the gripper FULLY CLOSED
  post-zero reading: +0.0058 rad
Step 2 — open the gripper to its mechanical limit
  full-open reading: 1.1486 rad (65.8°)
Write 1.1486 rad (65.8°) to MCU flash? [y/N] y
✓ stored: max_rad = 1.1486 rad (65.8°)
```

!!! warning "先摆好姿态,再按 Enter"
    固件在收到命令的瞬间锁存当时的原始计数。按 Enter 前夹爪必须**已经**在目标姿态
    (第 1 步完全闭合、第 2 步顶到机械极限),中途再动就白标了。

!!! tip "闭合恒为 0"
    零点写在固件里,**没有 `gripper_closed_rad` 配置**。`position_rad` 保持输出原始弧度,
    归一化只是新增 `position` 字段,不改变原有读数。

### 4.1.3 确认标定生效

**一、看启动日志。**采集程序连接时每侧会打印一行:

```text
[left]  Jaw normalised by the firmware's encoder-max calibration    ← 已生效
[left]  Firmware encoder-max calibration unavailable (...)          ← 未标定，走回退
```

**二、在 Rerun 里看曲线。**开 `--display_data=true`,标量面板里找 `gripper.pos`:

| 动作 | 期望 |
| --- | --- |
| 完全张开 | 顶到 **1.0** |
| 完全闭合 | 落到 **0.0** |

顶不到 1.0(例如停在 0.68)就是没标定,和日志的第二种情况对应。

### 4.1.4 适用范围

- **仅 leader(主夹爪)。**`Cmd::EncoderMaxCal` 是 leader 专有,follower 会 NACK
  `InvalidCmd`(它没有 MT6816 编码器)。follower 采集的数据两侧都走
  `gripper_open_rad` 回退,新旧算法一致。
- **需要固件 ≥ V2.1(leader 1.2.0)。**更低版本没有这条命令:`calibrate.py` 会**原样退出、
  不改动任何东西**,采集时则告警并自动退回旧算法——不中断会话,但标定不生效。
  **低于 V2.1 的夹爪必须先升级固件**,镜像随 SDK 附带 → [固件 OTA 升级](versions.md#ota)。
  **先升 SDK 再刷固件**,顺序反了会踩到一个"失败却报成功"的旧 bug。
- **标定是一次性的。**值写在 MCU flash 里,断电不丢,换主机不用重标。只有拆装编码器、
  更换机械限位或固件擦除后才需要重做。

!!! note "固件的上电自动标定不替代这一步"
    V1.9 引入了 `GripperAutoCalConfig`(上电时闭合到堵转⇒零点、张开到堵转⇒max_open),
    但该接口挂在 **follower** 上,leader 没有。采集用的是 leader,所以手动标定仍然必要。

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
**EEF TCP 位姿**(`tcp.*`)处绘制,并拖出一条走过的轨迹面包屑。

- 我们的位姿已在重力对齐世界系,场景声明为 `FLU`(X 前 / Y 左 / Z 上)——比只说明朝上轴的
  `RIGHT_HAND_Z_UP` 更完整,查看器因此知道哪个轴是"前",初始视角会朝 +X。世界轴带
  `+X forward / +Y left / +Z up` 标注,转动视角后仍读得出朝向。
- 机器人同时发布追踪器自身位姿时,Rerun 会**在 EE 坐标系旁再画一个更小更暗的追踪器坐标系**
  (仅显示,不落盘),两者之间连一条虚线并标出**长度(mm)**。这条长度是**判断安装变换对不对
  最快的一眼**:采集全程它应保持恒定;数值跳动或明显偏离机械尺寸,说明变换或装配有问题。
- `--show_trajectory` 默认开启;设为 `false` 可关闭。当 `--robot.enable_tracker=false`
  (无位姿可画)时会自动跳过。

!!! note "与 SDK 独立示例的区别"
    轨迹标记与面包屑的可视化形式和 SDK `python/examples/rerun_dual_with_tracker.py` 相似;正式 LeRobot 流程使用重力对齐的 `FLU` 世界系,SDK 独立示例展示 Pico4 原始 `LEFT_HAND_Y_UP` 坐标系。

标定与自检通过后,即可开始正式采集。

下一步 → [5. 数据采集](05-data-collection.md)
