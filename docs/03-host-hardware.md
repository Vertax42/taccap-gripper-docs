# 3. 主机与硬件配置

对应参考手册的"机器人配置"。夹爪能被**列出**不等于能被**打开**——本章解决串口权限、
ModemManager 抢占、设备自动发现规则,并给出硬件上电顺序。

!!! danger "[3.1 串口权限](#31) 与 [3.2 屏蔽 ModemManager](#32) 是必做项"
    两项都是**主机侧的一次性配置**,做完长期有效。

## 3.1 串口权限(dialout) {#31}

夹爪 MCU 枚举为 `/dev/ttyACM*`,属 `dialout` 组。若用户不在该组,SDK 能*列出*夹爪
但**打不开串口读固件序列号**,于是 `scan_grippers()` 报 `role=Unknown` / `firmware_sn`
为空,连接失败:

```text
RuntimeError: No leader gripper discovered for the left side.
# 底层实为: IoError: SerialBus: open(/dev/serial/by-id/...): Permission denied
```

一次性加入组,然后**重新登录**使其生效:

```bash
sudo usermod -aG dialout "$USER"
# 注销重登(或当前 shell 执行 newgrp dialout),然后重新插拔夹爪
```

验证——`role` 必须是 `Leader`/`Follower`(不是 `Unknown`),`firmware_sn` 非空:

```bash
python -c "from xense.taccap import scan_grippers
for g in scan_grippers(): print(g.side.name, g.role.name, repr(g.firmware_sn))"
```

!!! warning "序列号仍为空?"
    修好权限后 `firmware_sn` 仍为空,可能是 SN 未烧录、串口读取仍失败、固件通信异常或设备端配置问题;不能只根据空 SN 推断固件版本。保存完整报错并换线 / 换口复测,仍异常时联系设备或固件团队。

## 3.2 关闭 ModemManager 抢占(udev) {#32}

!!! info "用 Docker 交付镜像的话本节已经做过了"
    `install_customer.sh` 会把下面这条 udev 规则装到**主机**上(容器管不了主机的热插拔
    规则)。本节留作原理说明和排查参考,不用重做。

夹爪 MCU 是 CH343 USB 串口(`1a86:55d2`,CDC-ACM)。每次热插拔,**ModemManager**
(Ubuntu/GNOME 默认的蜂窝调制解调器服务)会用 AT 指令探测新端口并占用几秒,导致这段
时间内连接失败:

```text
IoError: SerialBus: open(/dev/serial/by-id/usb-1a86_USB_Dual_Serial_..-if02): Device or resource busy
```

!!! note "典型症状"
    **第一次**启动正常(端口已稳定),但拔下→换个 USB 口→立即重启就 **busy**。
    这**不是**触觉/相机/带宽问题。(若装了盲文驱动 `brltty`,它也会同样抢占 `1a86`。)
    临时办法:插好后等 ~3 秒再启动。

永久修复——用 udev 规则让 ModemManager 忽略这类设备(不影响真正的调制解调器):

```bash
sudo tee /etc/udev/rules.d/99-taccap-ignore-modemmanager.rules >/dev/null <<'EOF'
# XTac-UMI G1 MCUs are CH343 USB-serial (1a86:55d2) — keep ModemManager off them
ACTION=="add|change", SUBSYSTEMS=="usb", ATTRS{idVendor}=="1a86", ENV{ID_MM_DEVICE_IGNORE}="1"
EOF
sudo udevadm control --reload-rules && sudo udevadm trigger
```

验证:

```bash
udevadm info -q property -n /dev/ttyACM0 | grep ID_MM_DEVICE_IGNORE   # -> ID_MM_DEVICE_IGNORE=1
mmcli -L                                                               # 夹爪不再被列出
```

删除规则文件并重载即可还原。(专用机器人主机若无蜂窝模块,也可
`sudo systemctl disable --now ModemManager`。)

## 3.3 设备自动发现与"单左双右"规则 {#33}

所有设备**按序列号 + USB 拓扑自动发现**并分配到 `left`/`right`,**不手写序列号**。

### 序列号语法

| 设备 | 语法 | 示例 |
|---|---|---|
| 夹爪 Gripper | `TCGU01<batch><line><seq><m\|s>` | `TCGU01A24Z0002m` |
| 触觉 Tactile | `GSPS01<batch><line><seq>` | `GSPS01A25Z0011` |
| 相机 Camera | `XC<batch><line><seq><m\|s>` | `XCA24Z0007m` |

`<seq>` 为 4 位;patch `m` → leader(主夹爪),`s` → follower(从夹爪)。

### 侧别规则(单左双右)

4 位序列号的**最后一位**:**奇数 → 左,偶数 → 右**。适用于夹爪 / 腕相机的侧别,
以及同一只夹爪上**两枚指尖触觉传感器**的左右。

### 触觉左右 → `{side}_tactile_{left,right}`

结合 USB 拓扑与侧别规则:

- **属于哪个夹爪(side)**:共享同一夹爪 **USB hub** 的两枚 GSPS 传感器就是该夹爪的一对;
  夹爪的侧别读自其**固件 SN**(即 `scan_grippers()` 输出里的 side),**不是** CH343 的
  `mcu_serial`。即:hub → 夹爪 → 侧别。
- **哪一枚指尖触觉传感器(left/right)**:GSPS 序列号的**最后一位**(单左双右)。

### Pico4 Ultra 企业版追踪器——另一套序列号

追踪器序列号形如 `PC2310MLL3200496G`,末尾是字母 `G`。**根据 `G` 前的那个数字判断:单左双右**
——上面这个号 `G` 前面是 `6`,双数,所以这枚追踪器在**右**侧。这个 SN 需要从 PC Service 读取,
头显界面上看不到,见 [读取追踪器 SN](#pico-tracker-sn)。

!!! note "硬件烧错/装错会显式报错"
    遇到不合规序列号、每侧数量不对、两枚指尖触觉传感器映射到同一侧、两只夹爪抢同一触觉侧别,
    或触觉 hub 找不到对应夹爪时,设备发现会**直接报错并指明**出问题的 hub/序列号,
    避免实际装配和数据里的字段悄悄对不上。

!!! tip "先查「挤了两个」的那一侧"
    某侧为空,通常是它的设备**把自己算到了另一侧**去了。所以报错会先报重复、再报缺失——
    照着**挤了两个的那一侧**按上面的侧别规则复核序列号,比去找"不见了"的那只快。

## 3.4 Pico4 Ultra 企业版配置 {#34}

Pico4 Ultra 企业版配套的**独立运动追踪器**装在夹爪顶部,提供 6-DoF 位姿。在 Pico4 Ultra 企业版上运行 **XTac-UMI XR**
(VR 客户端 APP),位姿经 [XenseVR PC Service](#35) 送到采集端。
本节按**全新 Pico 从零配置**书写,顺序是「**开箱与系统更新 → 系统设置 → 安装 APP →
网络连接 → 绑定追踪器 → 追踪模式与界面设置 → 启动对齐**」。

!!! tip "拿到的是出厂已配置的头显?跳过前面几节"
    开发者模式、电源策略、APP 安装、追踪器绑定、追踪模式这些都已在出厂时设好,写在头显里
    断电不丢——凡是标着「**预配置设备可跳过本节**」的都不用看。

    **直接从 [网络连接](#pico-network) 开始**:接上 USB、短按追踪器电源键到蓝灯亮,
    然后照 [界面清单](#pico-toolkit-ui) 核对四项、做 [启动对齐](#pico-frame)。这三步
    **每次采集都要做**,预配置与否都一样。

### 开箱与系统更新 {#pico-unbox}

!!! info "预配置设备可跳过本节"
    本节的设置写在头显里,断电不丢。出厂已配置的设备无需重做——除非恢复出厂设置或更换头显。

**1. 开箱**:撕掉机身前面板保护贴、控制器胶条与镜片贴纸,长按电源键开机。

=== "崭新包装"

    ![PICO 4 Ultra 企业版原包装](assets/pico4/unbox-box.jpg){ width="440" }

=== "撕前面板保护贴"

    ![头显前面板传感器条上的保护贴](assets/pico4/unbox-film.png){ width="440" }

=== "撕完的样子"

    ![保护贴撕除后的头显](assets/pico4/unbox-film-done.jpg){ width="440" }

=== "撕控制器胶条"

    ![控制器上的固定胶条](assets/pico4/unbox-controller-tape.png){ width="440" }

=== "撕镜片贴纸"

    ![镜片(眼罩)上的保护贴纸](assets/pico4/unbox-lens-sticker.jpg){ width="440" }

=== "长按开机"

    ![长按电源键开机](assets/pico4/unbox-power-on.png){ width="440" }

**2. 更新系统**——**新机必做**:连上**有网的 WiFi**,进 **设置 → 系统升级**,
点「**下载并安装**」升到 **5.15.5.U 或更高**。安装包约 1.9 GB,留足下载与重启的时间。

![设置 → 系统升级:检测到 5.15.5.U](assets/pico4/system-update.jpg){ width="560" }

!!! danger "新机必须先升级系统"
    出厂系统版本偏低,**升级之后才能正常使用**。请在配对追踪器、装 APP 之前先做完这一步。

### 系统设置:开发者模式与电源策略 {#pico-system}

!!! info "预配置设备可跳过本节"
    本节的设置写在头显里,断电不丢。出厂已配置的设备无需重做——除非恢复出厂设置或更换头显。

**1. 开启开发者模式**:在 Pico4 Ultra 企业版内打开 设置 → 关于本机 → 连续点击「软件版本号」数次 →
左侧出现「开发者选项」→ 打开 **USB 调试**。

=== "打开设置"

    ![资源库 → 设置](assets/pico4/devmode.png){ width="480" }

=== "连点软件版本号"

    ![关于本机 → 连续点击软件版本号](assets/pico4/devmode-tap-version.png){ width="480" }

=== "开发者选项 + USB 调试"

    ![左侧出现开发者选项,打开 USB 调试](assets/pico4/devmode-usb-debug.png){ width="480" }

**2. 关闭休眠与灭屏**:开发者模式打开后,进入 企业设置 → 系统设置 → **电源策略**,
按下面的**顺序**把两项都改成「**永不**」:

1. 先设 **系统休眠 = 永不**;
2. 再设 **灭屏(息屏)= 永不**;
3. 顺手把最下方的**电量及充电状态图标**设为「**常驻显示**」——采集途中能随时看到剩余电量。

=== "进企业设置"

    开发者选项里点「**企业设置**」。

    ![开发者选项 → 企业设置](assets/pico4/power-step1-enterprise.jpg){ width="480" }

=== "系统设置 → 电源策略"

    ![企业设置 → 系统设置 → 电源策略](assets/pico4/power-step2-policy.jpg){ width="480" }

=== "改之前的默认值"

    出厂默认是**灭屏 30 秒、系统休眠 5 分钟、电量图标不显示**——三项都要改。

    ![电源策略的出厂默认值](assets/pico4/power-step3-before.jpg){ width="480" }

=== "最终设置"

    ![电源策略:灭屏与系统休眠均为「永不」,电量图标常驻显示](assets/pico4/power-step4-final.png){ width="480" }

改完应该和「**最终设置**」那张一致:**灭屏 = 永不、系统休眠 = 永不、电量及充电状态图标 = 常驻显示**。

!!! warning "顺序不能反"
    灭屏时间受系统休眠时间约束。若先改灭屏,系统休眠仍是默认值,灭屏的「永不」会被
    钳制回一个有限值(表面看已设置,实际未生效)。**先休眠、后灭屏**,改完退出设置再回来
    确认两项都显示「永不」。

不做这一步的后果:头显在采集间隙自动灭屏/休眠后,**XTac-UMI XR 会被系统挂起或杀掉**,
追踪数据中断;重启 XTac-UMI XR 又会重新冻结世界系原点与方向,导致同一数据集内位姿参考系不一致
(见 [坐标系对齐](#pico-frame))。摘下头显放置时同样会触发,所以必须关掉,不能只靠「别摘头显」。

!!! note "本项在企业设置里,不在普通设置里"
    「电源策略」只在 **Pico 企业版**的「企业设置」中提供;消费版设置菜单里没有这一项,
    也无法把灭屏调到「永不」。

### 安装 XTac-UMI XR(Pico4 Ultra 企业版) {#pico-app}

!!! info "预配置设备可跳过本节"
    本节的设置写在头显里,断电不丢。出厂已配置的设备无需重做——除非恢复出厂设置或更换头显。

=== "拷贝 apk(电脑侧)"

    用 USB 线连接 Pico4 Ultra 企业版与电脑,把 `XTac-UMI-XR-0.1.X.apk` 拷到头显的
    `Download/` 目录(`0.1.X` 是版本号,以你拿到的那份为准)。

    ![电脑侧把 apk 拷进 Pico 的 Download 目录](assets/pico4/install-step1-copy.jpg){ width="480" }

=== "在头显里找到它"

    戴上头显,任务栏点「**文件管理**」,进「**Download**」文件夹。

    ![文件管理 → Download](assets/pico4/install-step2-filemanager.png){ width="480" }

=== "安装并确认"

    点 `XTac-UMI-XR-0.1.X.apk`,在弹出的「要安装此应用吗?」里选「**安装**」。

    ![确认安装](assets/pico4/install-step3-confirm.png){ width="480" }

    装好后回到「**资源库**」,应该能看到名为 **XTac-UMI XR** 的应用。

    ![安装包里的 apk,以及装好后出现在资源库里的 XTac-UMI XR](assets/pico4/install-step3-done.jpg){ width="480" }

### 网络连接(重要) {#pico-network}

追踪数据要送到数采电脑上的 XenseVR PC Service,**有线和无线两种接法都支持**:

| 接法 | 说明 |
|---|---|
| **USB 有线共享网络**(推荐) | 用 Type-C 线直连数采电脑,由头显给电脑分配 IP |
| WiFi 无线 | 头显与数采电脑接入**同一个网络**,不用接线 |

!!! tip "采集时优先走有线"
    **WiFi 环境复杂时(信道拥挤、干扰多)无线传输会受影响**,表现为位姿卡顿、抖动或掉数据,
    而且不容易和别的问题区分。有线共享网络更稳定,正式采集建议用它;无线适合临时调试或
    不方便接线的场合。

**有线连接步骤:**

1. 在 Pico4 Ultra 企业版内打开 设置 → 开发者选项 → 打开「USB 调试」→「USB 连接」选择「**传输文件**」。
   **每次拔插 USB 后都要回来确认这一项**——它会掉回默认值。选不了就重启 Pico 再试。
2. 打开 **XTac-UMI XR** → 勾选 **"shared network (connect USB first)"** → 等待 Pico4 Ultra 企业版为
   PC 端分配 IP → 输入 **PC Service 的 IP** 连接。
3. 电脑端启动服务(见 [§3.5](#35)):`runService.sh`。

走 **WiFi** 时不用勾 `shared network`(那一项是给 USB 共享网络用的):两端连上同一网络后,
在同一处填 **PC Service 的 IP** 即可,第 3 步相同。

!!! danger "走有线时,关掉数采电脑的 WiFi"
    有线共享网络会与电脑上的**其他网络(尤其 WiFi)冲突**(路由 / 网卡抢占),导致追踪器
    连不上或位姿不稳。**用有线就把电脑的 WiFi 关掉**,只保留头显的共享网络。

![USB 连接选择传输文件](assets/pico4/usb-shared-network.jpg){ width="520" }

### 绑定运动追踪器到头显 {#pico-tracker-bind}

!!! info "预配置设备可跳过本节"
    本节的设置写在头显里,断电不丢。出厂已配置的设备无需重做——除非恢复出厂设置或更换头显。

**首次使用、或更换追踪器后**,必须先把 PICO Motion Tracker **绑定到这台头显**。
未绑定时,追踪模式里选不到它,XTac-UMI XR 与 PC Service 也发现不了对应 SN。

!!! tip "配对前先用手机扫追踪器背面的二维码拿到 SN"
    扫出来的就是这枚追踪器的完整 SN,配对时可以直接核对,不用等装好了再去 PC Service 查。
    **装的时候按 SN 对应侧别**(单左双右,见 [3.3](#33)):**单数装左夹爪,双数装右夹爪**。

    | 扫这里 | 扫出来是这样 |
    |---|---|
    | ![追踪器背面的二维码](assets/pico4/tracker-sn-qr.jpg){ width="300" } | ![左追踪器的扫描结果](assets/pico4/tracker-sn-left.png){ width="320" }<br>`G` 前是 `1`,单数 → 装**左**夹爪<br><br>![右追踪器的扫描结果](assets/pico4/tracker-sn-right.png){ width="320" }<br>`G` 前是 `8`,双数 → 装**右**夹爪 |

    红框圈出的六位数,就是配对后「**我的追踪器**」列表里显示的编号(如 `Tracker 150311`)——
    列表里哪一项对应哪一枚,靠这六位数对上。

1. 从 **资源库** 打开「**体感追踪器**」App,进入**配对界面**。
2. **长按追踪器电源键约 6 秒**,直到指示灯**蓝红交替闪烁**——这是蓝牙配对状态
   (只亮蓝灯是普通开机,不是配对状态,App 扫不到)。
3. 在配对界面点「**开始配对**」,等待绑定完成。**配对成功时头显会响一声提示音**——听到提示音
   就表示绑上了。该追踪器随即出现在「**我的追踪器**」列表里,显示电量与编号
   (如 `Tracker 150399`)并标注「**已连接**」。
4. **两只夹爪各一枚,需要绑定两枚**。列表顶部应显示「**已配对 2 个**」。

!!! tip "开机是短按,配对才要长按"
    **日常开机**只需**短按**电源键,蓝灯亮起即可。
    **只有首次绑定**才长按约 6 秒到**蓝红交替闪烁**——平时别按到进配对状态。

=== "打开体感追踪器 App"

    从**资源库**里点开「**体感追踪器**」。

    ![资源库 → 体感追踪器](assets/pico4/tracker-app-open.png){ width="440" }

=== "进入配对界面"

    在「PICO 体感追踪器」主界面,点**右上角的图标**进入配对界面。

    ![体感追踪器主界面,右上角进入配对](assets/pico4/tracker-pair-entry.png){ width="440" }

=== "我的追踪器:已配对 2 个"

    ![体感追踪器 App:两枚追踪器均已连接](assets/pico4/tracker-bind.jpg){ width="440" }

!!! note "绑定是一次性的;解除配对在 ⓘ 里"
    绑定关系保存在头显上,日常开关机、重启 APP **不需要重绑**;
    更换追踪器、换用另一台头显或头显恢复出厂设置后需要重新绑定。
    **更换设备**时先在列表项右侧的 **ⓘ** 中解除配对,再绑新的。

!!! warning "独立追踪模式下,追踪器必须在头显视野内"
    App 自身也会提示这一点。追踪器被身体、桌沿或另一只手长时间遮挡会**丢跟踪**,
    表现为位姿跳变或卡住。采集时注意作业姿态,别让夹爪顶部的追踪器长时间脱离头显视野。

#### 读取追踪器 SN {#pico-tracker-sn}

SN 决定左右(末尾字母 `G` 前一个数字:单左双右,见 [3.3](#33)),也是 PC Service 识别追踪器的依据。

头显里看不到这个 SN:「体感追踪器」App 只显示**短编号**(如 `Tracker 150399`),
XTac-UMI XR 的 Network 面板显示的 SN(如 `PA9410MGL…`)是**头显自己的**。
匹配左右要用的**追踪器完整 SN**(形如 `PC2310MLL3200496G`)用 PC Service 的 Python 接口
`xensevr_pc_service_sdk` 读取:

```python
import xensevr_pc_service_sdk as xrt

xrt.init()
print(xrt.get_motion_tracker_serial_numbers())   # 例:['PC2310MLL3200496G', ...]
```

!!! warning "读 SN 需要整条链路先跑起来"
    `get_motion_tracker_serial_numbers()` 报的是**服务当前收到数据的**追踪器。
    所以要先:追踪器已绑定并开机 → XTac-UMI XR 显示[「已连接」](#pico-toolkit-ui)
    → 主机已启动 [XenseVR PC Service](#35)。少任一步,返回的会是空列表。

拿到 SN 后可用 `--robot.tracker_serial=<SN>` 直接钉住,跳过自动匹配。
**逐个摇晃夹爪**确认哪个 SN 对应哪只手,再写进配置。

### 追踪模式与 XTac-UMI XR 设置 {#pico-tracker}

!!! info "预配置设备可跳过本节"
    本节的设置写在头显里,断电不丢。出厂已配置的设备无需重做——除非恢复出厂设置或更换头显。

绑定完成后:

1. 在 Pico4 Ultra 企业版里打开「**体感追踪**」。
2. 进入设置,追踪模式选「**独立追踪**」。

=== "进设置找到「追踪模式」"

    出厂默认是「**全身动捕**」——把追踪器戴在身上追人体姿态,不是我们要的。

    ![设置 → 追踪模式](assets/pico4/tracker-mode1-entry.png){ width="480" }

=== "选「独立追踪」"

    「**独立追踪**」是把追踪器固定在物体上追物体的位姿——夹爪就属于这种用法。
    选中后点「**确定**」。

    ![追踪模式:选独立追踪并确定](assets/pico4/tracker-mode2-pick.png){ width="480" }

=== "改完的样子"

    「追踪模式」这一行应该显示「**独立追踪**」。

    ![设置里追踪模式已是独立追踪](assets/pico4/tracker-mode3-done.png){ width="480" }


追踪器由 XenseVR PC Service 按**序列号(SN)**识别;侧别按 SN 末尾字母 `G` 前一个数字单左双右自动匹配
(见 [3.3](#33)),或用 `--robot.tracker_serial=<SN>` 直接钉住。

### 打开 App 后的界面 {#pico-toolkit-ui}

戴上头显,从**资源库**打开 **XTac-UMI XR**。界面很简单,只有**状态**、**分辨率**、
**重连**三项(左边的「折叠」把面板收起来)。**要的就是「状态」显示「已连接」**——
在那之前 PC 端读不到任何位姿。

**不用填 PC 端 IP**:[有线共享网络](#pico-network)接好后,APP 会自动识别并连上主机上的
XenseVR PC Service。

=== "打开软件"

    从**资源库**点开 **XTac-UMI XR**。

    ![资源库 → XTac-UMI XR](assets/pico4/app-step1-open.png){ width="480" }

=== "状态:未连接"

    刚打开时通常显示「**状态:未连接**」,等一下就会自己连上;没连上就点「**重连**」。

    ![XTac-UMI XR:状态 未连接](assets/pico4/app-step2-disconnected.jpg){ width="420" }

=== "允许相机权限"

    首次打开会问「允许"XTac-UMI XR"使用相机权限吗?」,点「**允许**」。
    [头显相机](05-data-collection.md#57)要用到它,点了「不允许」就取不到头显画面。

    ![允许 XTac-UMI XR 使用相机权限](assets/pico4/app-step3-camera.png){ width="380" }

=== "状态:已连接"

    「**状态:已连接**」就是可以开始采集的状态。

    ![XTac-UMI XR:状态 已连接](assets/pico4/app-step4-connected.jpg){ width="420" }

!!! tip "「分辨率」是头显双目相机的分辨率"
    指的是[头显相机](05-data-collection.md#57)的取流分辨率,默认 `1024`,
    不用头显相机时它不起作用。没有特殊需要保持默认即可。

!!! note "高精度追踪已默认常开"
    高精度追踪模式(位姿更稳、抖动更小)现在默认开启,界面上没有开关,不需要手动设置。

!!! warning "一直连不上?先查网络这一步"
    点了「重连」仍是「未连接」,问题多半不在 APP,而在
    [网络连接](#pico-network):有线共享网络没接好,或数采电脑的 WiFi 没关。

!!! tip "自检:PC 端有没有真的收到"
    显示「已连接」之后,在主机上用 `/opt/apps/roboticsservice/` 的 `ConsoleDemo` 或
    `python -m lerobot.robots.taccap_gripper.calibrate_tracker` 确认能读到带 `sn` 的位姿——
    头显里显示连上,和主机真的收到数据,是两件事。

### 启动与坐标系对齐 {#pico-frame}

**佩戴 Pico4 Ultra 企业版启动 XTac-UMI XR 时,面朝机器人正前方**,再按
[界面清单](#pico-toolkit-ui)确认状态显示「**已连接**」。启动瞬间
**冻结世界系的原点与方向**。

录制位姿落在**重力对齐的世界系**:**X 正 = 面朝前方,Y 正 = 左,Z 正 = 上**。

!!! warning "启动时必须面朝机器人正前方"
    面朝正前方,世界系 X 轴正方向才对齐机器人正前方(Y 左、Z 上)。**只需对齐方向**;
    站的位置(原点)不影响后续使用。

!!! warning "分集之间不要重启 XTac-UMI XR"
    一旦重启,后续录制的原点/方向会变,导致同一数据集内位姿参考系不一致。

- **Pico4 Ultra 企业版原始系**:左手系(X 右、Y 上、Z 里),原点 = 启动时 Pico4 Ultra 企业版 Head 位置。
- **录制系**:采集时会重映射到上述世界系(X 前、Y 左、Z 上)。

## 3.5 启动 XenseVR PC Service {#35}

追踪器与主机的 **XenseVR PC Service**(RoboticsService)守护进程通信;它负责设备发现、
状态监控与实时追踪数据分发,采集端从它读取位姿。

!!! info "用 Docker 交付镜像的话服务会自动启动"
    容器启动时会自己拉起这个服务,不用手动执行下面的命令。只处理数据、用不到追踪器时,
    可以用 `START_XENSEVR_SERVICE=0` 关掉(见 [2. 环境部署 · Docker](02-environment.md#docker))。

启动:

```bash
/opt/apps/roboticsservice/runService.sh
```

!!! note "同一时间只能运行一个实例"
    该服务一次只允许运行一个实例;重复启动会失败或冲突。

服务可提供多类追踪数据(Pico4 Ultra 企业版 Head / 手柄 / 手势 / 全身动捕 / **Tracker 独立追踪**);数采使用的是
**Tracker 独立追踪**位姿,数据中带 `sn` 用于区分不同追踪器。

!!! note "头显相机也走这个服务(需要 v0.2.0)"
    [头显相机](05-data-collection.md#57)的画面不是另一条链路:头显把每只眼的画面发给
    PC Service,服务转发给采集端。所以**头显相机和追踪器共用同一个服务、同一条连接**——
    服务没起来,两者都没有。

    转发画面需要 **v0.2.0 以上**(见 [2.4 一键安装](02-environment.md))。
    只用追踪器时,版本没有区别。

!!! tip "验证服务与设备"
    服务目录附带 `ConsoleDemo` / `RobotDemoQt` 演示程序(`/opt/apps/roboticsservice/`),
    可用于确认 Pico4 Ultra 企业版已被发现、追踪数据正常(需与服务相同的运行环境)。

## 3.6 硬件上电顺序 {#36}

!!! note "启动顺序"
    标准启动顺序如下(APP 界面以实际版本为准):

1. 将 XTac-UMI G1 插入主机(USB)。
2. 接好 Pico4 Ultra 企业版的**有线共享网络**,并**关闭数采电脑的 WiFi**(见 [3.4 网络连接](#pico-network))。
3. 开启 Pico4 Ultra 企业版,**短按**追踪器电源键至**蓝灯亮起**(首次使用需先[绑定](#pico-tracker-bind))。
4. **面朝机器人正前方**,启动 XTac-UMI XR APP(**冻结世界系原点与方向**,见 [坐标系](#pico-frame)),
   并确认 APP 的[状态显示「已连接」](#pico-toolkit-ui)。
5. 启动主机的 XenseVR PC Service(`runService.sh`)。
6. 运行标定 / 自检 / 录制脚本。

```mermaid
flowchart LR
    A[插入夹爪 USB] --> N[接 Pico4 Ultra 企业版<br/>有线网络并关闭 WiFi]
    N --> B[开启 Pico4 Ultra 企业版<br/>配对追踪器]
    B --> C[启动 XTac-UMI XR<br/>冻结原点]
    C --> D[启动 XenseVR PC Service]
    D --> E[跑标定/录制]
```

!!! warning "主夹爪没标定的话,采集程序会拒绝连接"
    数据集里的 `gripper.pos` 是归一化开度(`0.0` 闭合 / `1.0` 张开),这两个端点来自写在
    MCU flash 里的**编码器零点**和**行程上限**。**没标过行程上限的主夹爪连不上**,程序会
    带着标定命令报错退出——所以不用自己判断该不该标,能连上就是标过了。

    标定值存在 flash 里,断电不丢、换主机也不用重标,一台标一次就够,不是每次开录的例行步骤。

    双夹爪尤其注意:**只标一侧会让左右两条通道落在不同刻度上**,同一个握持动作左右读数不同,
    而数据里看不出任何异常。要标就两侧都标:

    ```bash
    python third_party/taccap-gripper/python/examples/calibrate.py left
    python third_party/taccap-gripper/python/examples/calibrate.py right
    ```

    完整步骤、如何确认生效、适用范围 → [4.1 夹爪标定(零点 + 行程)](04-calibration.md#41)

下一步 → [4. 标定与自检](04-calibration.md)
