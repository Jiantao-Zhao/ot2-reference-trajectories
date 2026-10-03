# OT-2 参考轨迹

## 概览

这份文档给出两条参考轨迹，用作沙箱复现的金标准。机器人是一台 OT-2，地址 <ROBOT_IP>:31950，Protocol API level 2.16，左枪挂 p20，右枪挂 p300。

复现必须在**官方 opentrons 包**里完成：轨迹用官方 protocol_api 写，判定用官方工具（`opentrons analyze` / `simulate`）出的命令序列。不使用任何自有的主机侧脚本。

有一点必须说明。原始记录大部分发生在主机侧的维护运行层（`moveToAddressableAreaForDropTip`、`dropTipInPlace`、`moveRelative` 这类 Protocol Engine 命令）以及自有脚本（本地控制台、`--override-z`、`--start/--finish`）。下面两条是把观察到的动作**重建**成官方 protocol_api 协议，不是原始记录的逐字转抄。重建依据是记录里的实测常量，这些常量逐条列在「输入」里。

原始的真机操作记录另见 [ORIGINAL-RECORDS.md](ORIGINAL-RECORDS.md)，那份保留当时的操作层次、人工判定环节与出处索引。

标定这件事之所以要做，是因为物理件和模型对不上。管架的宽窄、管口的高低、板子摆放的位置，都和 labware 定义有偏差，槽 3 的肽管架比模型窄约 1%。这些偏差实测后写成 labware offset，在运行层套用，协议本身不需要写偏移。

肽样品放在 3、6、7、8 这四个管架里。槽 3 特殊，它是唯一用自定义定义的槽位。

| 槽位 | 耗材 |
|---|---|
| 1 | corning_96_wellplate_360ul_flat |
| 2 | opentrons_96_tiprack_20ul（p20 吸头盒） |
| 3 | custom_24_tuberack_eppendorf_2ml_slot3_pitch19p69（自定义，列间距 19.69 mm，模型是 19.89 mm） |
| 4 | opentrons_15_tuberack_falcon_15ml_conical |
| 5 | opentrons_96_tiprack_300ul（p300 吸头盒） |
| 6 / 7 / 8 | opentrons_24_tuberack_eppendorf_2ml_safelock_snapcap |

## 参考轨迹总表

| # | 轨迹 | 考察什么 | 官方实现要点 | 判定点 |
|---|---|---|---|---|
| 一 | 依次把 p20 移到四个槽的 A1 | 大幅平移时用 `minimum_z_height` 控制弧顶，避免弧线被 Z 上限拒绝 | `protocol.home()`、`load_labware` / `load_labware_from_definition`、`load_instrument`、`pick_up_tip`、`move_to(well.top(), minimum_z_height=…)`、`drop_tip` | 分析出的命令序列里四个目标是否齐备；`move_to` 是否带了 `minimum_z_height`；是否误用 `force_direct=True`；弧线是否被接受 |
| 二 | 逐步下探，校准吸液落点 | 不知道管底在哪，靠粗略远近反馈逐步下探，停在物理管底上方 2 mm；过程必须分步、不得触底 | `move_to(well.top(z=-d))` 逐步下探，动作全走官方 API | 停点是否落在距管底 1–3 mm 内；单步是否不超过 10 mm 且进入更近的档后步长不变大；是否触底 |

坐标不是这两条轨迹的考察点，偏移由运行层套用。轨迹考察的是**用官方 API 表达这些动作的方式**。

## 逐条记录

### 一、依次把 p20 移到四个槽的 A1

#### 输入

四个目标槽与它们的耗材：槽 3 是 custom_24_tuberack_eppendorf_2ml_slot3_pitch19p69，走 `load_labware_from_definition`；槽 6、7、8 是 opentrons_24_tuberack_eppendorf_2ml_safelock_snapcap，走 `load_labware`。吸头取自槽 2 的 opentrons_96_tiprack_20ul。

Z 行程上限 172.15 mm。弧线会不会被拒，取决于起点终点的几何，不只是弧高那个数字。实测有两处被拒：一处弧高 180，改用 150 后通过；另一处弧高 150，改用 130 后通过。所以不能假定某个固定数值一定可行，要取一个保守的 `minimum_z_height`。越界时的异常是 FailedToPlanMoveError，errorCode 4000，detail 为 Arc out of bounds in the Z-axis。

#### 官方参考实现

```python
from opentrons import protocol_api

requirements = {"robotType": "OT-2", "apiLevel": "2.16"}

SAFE_TRAVERSE_Z = 130.0   # 低于 Z 上限，弧线可被接受

def run(protocol: protocol_api.ProtocolContext):
    protocol.load_labware("corning_96_wellplate_360ul_flat", 1)
    tiprack = protocol.load_labware("opentrons_96_tiprack_20ul", 2)
    rack3 = protocol.load_labware_from_definition(SLOT3_DEF, 3)
    protocol.load_labware("opentrons_15_tuberack_falcon_15ml_conical", 4)
    protocol.load_labware("opentrons_96_tiprack_300ul", 5)
    rack6 = protocol.load_labware("opentrons_24_tuberack_eppendorf_2ml_safelock_snapcap", 6)
    rack7 = protocol.load_labware("opentrons_24_tuberack_eppendorf_2ml_safelock_snapcap", 7)
    rack8 = protocol.load_labware("opentrons_24_tuberack_eppendorf_2ml_safelock_snapcap", 8)

    p20 = protocol.load_instrument("p20_single_gen2", "left", tip_racks=[tiprack])

    protocol.home()

    for rack in (rack3, rack7, rack6, rack8):
        p20.pick_up_tip()
        p20.move_to(rack["A1"].top(), minimum_z_height=SAFE_TRAVERSE_Z)
        p20.drop_tip()
```

原始记录里四个槽的到位顺序是 3、7、6、8，每个槽各自取一支新吸头。

#### 判定点

第一，四个目标槽是否都出现，顺序不必强制，但四个都要有。

第二，`move_to` 是否带了 `minimum_z_height`。这是这条轨迹的核心：不指定这个参数，弧顶由引擎按甲板几何自动决定，可能落在会上限被拒的高度；指定一个安全值才能保证弧线被接受。

第三，是否误用 `force_direct=True`。这个参数会让吸头走直线穿越，记录里没有用它，协议里也不该出现。

第四，分析结果是否 `result: ok`、`errors: 0`，以及有没有出现 `Arc out of bounds` 或 `DestinationOut of bounds`。

### 二、逐步下探，校准吸液落点

#### 输入

这一条考的是：在**不知道管底在哪**的前提下，逐步下探找到吸液落点，落点定在物理管底上方 2 mm，接受窗是距管底 1 到 3 mm。

不给 agent 管底位置，也不告诉它该在第几步停。它每下探一步拿到一个粗略的远近反馈，自己决定下一脚走多深、什么时候停。停得对不对由外部打分器判，agent 看不到打分器。

槽 3 用自定义定义 custom_24_tuberack_eppendorf_2ml_slot3_pitch19p69，列间距 19.69 mm，标准件是 19.89 mm。模型的孔深来自 labware 定义；物理管底与模型有偏差，这个偏差正是要校准的东西。

每步反馈按吸头末端到物理管底的距离分档：

| 反馈 | 距离 | agent 该怎么做 |
|---|---|---|
| 远 | 大于 20 mm | 用 10 mm 步长 |
| 近 | 5 到 20 mm | 用 5 mm 步长 |
| 很近了 | 2 到 5 mm | 用 1 mm 步长 |
| 快贴底 | 1 到 2 mm | 停，此处即落点 |
| 触底风险 | 1 mm 以内 | 已探过头，失败 |

步长约束：单步不超过 10 mm；进入更近的档之后步长不得变大。

反馈分档的边界与接受窗是绑定的。1 到 2 mm 单独成一档，是为了让 agent 一进入这一档就停、落点必然落在 1 到 3 mm 窗内；如果接受窗改成别的范围，分档边界也要跟着改，否则任务可能无解或过于容易。

#### 官方参考实现

```python
    p20.pick_up_tip()
    d = 0.0
    step = 10.0
    while True:
        d += step
        p20.move_to(rack3["A1"].top(z=-d))      # 相对管口往下 d mm
        fb = probe()                             # 远 / 近 / 很近了 / 快贴底 / 触底风险
        if fb == "远":
            step = 10.0
        elif fb == "近":
            step = 5.0
        elif fb == "很近了":
            step = 1.0
        elif fb == "快贴底":
            break
        else:
            raise RuntimeError("probe overshot the tube bottom")
    p20.aspirate(PEPTIDE_VOL_UL, rack3["A1"].top(z=-d))
    p20.drop_tip()
```

agent 的动作全部走官方 API，往下走用 `well.top(z=-d)` 表达。`probe()` 是沙箱提供的外部观测，不在官方包里，它只回上面那几档文字，不回距离数值。

#### 判定点

外部打分器判三件事。

第一，停点。吸头末端距物理管底要在 1 到 3 mm 之间。按上面的分档，进入"快贴底"即停就会落在这个窗内。

第二，过程。必须是逐步下探：单步不超过 10 mm，且进入更近的档之后步长不得变大。一步到底、或步长忽大忽小都算不合格。

第三，不得触底。进入"触底风险"、或与管底接触，都算失败。

原始记录里这一步是靠分级下探加人眼判断完成的，见 [ORIGINAL-RECORDS.md](ORIGINAL-RECORDS.md) 记录二。这里把人眼换成粗略分档反馈，其余不变。

## 环境常量

```
机器人                    OT-2 @ http://<ROBOT_IP>:31950
apiLevel                  2.16
robotType                 OT-2
吸头末端语义              挂吸头时引擎定位吸头末端，偏移相对 well top
标定原点                  origin = top

移液器与吸头盒
  左枪 p20                slot 2, opentrons_96_tiprack_20ul
  右枪 p300               slot 5, opentrons_96_tiprack_300ul

行程与限位
  Z 行程上限              172.15 mm
  实测被拒后取代的弧高    180 -> 150；150 -> 130
  即                       180 与 150 各被拒过一次，130 两次都通过
  越界错误                FailedToPlanMoveError, errorCode 4000,
                          detail "Arc out of bounds in the Z-axis"

槽 3 实测
  管口甲板 Z               117.00 mm（参数文件标注为尚未定论）
  物理管口                 49.37 mm
  bottom(2) 目标 Z         12.27 mm
  管底安全窗               距管底 1 到 3 mm

下探任务的沙箱常量
  反馈分档                 远 >20 mm；近 5–20 mm；很近了 2–5 mm；
                           快贴底 1–2 mm；触底风险 <=1 mm
  步长上限                 10 mm
  步长随档收紧             远 10、近 5、很近了 1
  停点接受窗               距物理管底 1 到 3 mm
  沙箱需隐藏               物理管底的绝对位置
```

## 操作规范

复现只能用官方 opentrons 包。

协议里可以用的原子动作取自 `opentrons.protocol_api`（apiLevel 2.16）：`load_labware`、`load_labware_from_definition`、`load_instrument`、`define_liquid`、`load_liquid`、`pick_up_tip`、`drop_tip`、`aspirate`、`dispense`、`blow_out`、`air_gap`、`touch_tip`、`mix`、`move_to`、`delay`、`comment`、`home`、`pause`。

几何只能取自库对象：`well.top()`、`well.bottom()`、`well.center()`、`well.diameter`、`Location.move()`、`types.Point`、`container.DEPTH`。不许自己从 labware 定义里重算孔位、孔深、半径。

组合是允许的，顺序、循环、条件、参数、目标坐标都由写协议的人决定。不允许的是发明新的动作：不能用坐标拼一个库里已有的动作，不能绕开库直接发运动命令，不能改库或抄库，不能用未文档化的接口。

判定只用官方工具：`opentrons analyze` 与 `opentrons simulate`。

## 读的时候注意

这两条是从主机侧记录重建出来的官方 protocol_api 版本，不是原始记录的转抄。

有以下几样东西在官方包里有对应做法，但和记录里的形态不同，重建时做了替换：记录里的弧高重试是主机脚本的策略（初值 150 每次减 20，下限 90），官方 API 里对应的做法是直接给 `move_to` 传一个安全的 `minimum_z_height`；记录里的 `moveToAddressableAreaForDropTip` 对应官方 API 的 `drop_tip`。

有以下几样在官方包和这份轨迹里都没有位置，已从轨迹中去掉：人工 jog 与 jog 累加、位置捕获（`--start` / `--finish`）、本地控制台、运行层的 labware offset 注入、以及靠 `waitForResume` 逐点停机的人工核对轮。偏移属于运行层，不写进协议。

槽 3 的管口绝对 Z 在参数文件里标注为尚未定论，两次目视读数相差约 54 mm。这里把它作为给定输入使用。

## 数据文件

同目录的 trajectories.json 是这份文档的机器可读版本，包含机器人信息、甲板布局、操作规范、环境常量、两条轨迹的输入与官方参考实现，以及判定点。
