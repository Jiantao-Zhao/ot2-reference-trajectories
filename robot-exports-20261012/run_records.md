# 运行记录

这份文件汇总从历史运行里能取到的部分。**注意先看第一节：完整的命令记录已经取不到了。**

所有条目来自本地会话日志（工具调用与返回），不是从机器实时导出的。

## 一、完整的命令记录取不到

这台机器的维护运行用完即弃。2026-09-30 逐个探测过 7 个 run，`/runs/{id}` 与 `/runs/{id}/commands` **全部返回 404**：

```
565a5d1c  status=404  commands=None
ac20a023  status=404  commands=None
884ad0dd  status=404  commands=None
f340314c  status=404  commands=None
bb172223  status=404  commands=None
cf1b1b32  status=404  commands=None
e6d37afb  status=404  commands=None
```

另外这台机器的 API（26.6.0）没有实时位置接口（`/robot/state`、`/robot/settings` 等也是 404），命令级参数只在**当时的 dry-run 输出**里留下过。

所以请求单第 4 项（`ac20a023` 的完整记录和命令）只能给出下面的摘要，拿不到逐条命令的完整参数。

## 二、五次弧高的观测（这是关键部分）

| # | 槽 | 弧高 (mm) | 结果 | 命令 | 备注 |
|---|---|---|---|---|---|
| 1 | 3 | **180** | 被拒 | `moveToAddressableAreaForDropTip` | 首次尝试 |
| 2 | 3 | **150** | **通过** | 同上 | 降低弧高后重试 |
| 3 | 7 | **150** | 被拒 | 同上 | 首次尝试 |
| 4 | 7 | **130** | 通过 | 同上 | 降低弧高后重试 |
| 5 | 6 | **130** | 通过 | 同上 | 首次尝试即为 130 |

两次被拒的完整错误对象：

```json
// 槽 3，弧高 180
{"id": "ac23d822-32a7-4aef-a734-5b216822acee",
 "createdAt": "2026-09-30T08:35:40.840033Z",
 "isDefined": false,
 "errorType": "FailedToPlanMoveError",
 "errorCode": "4000",
 "detail": "Arc out of bounds in the Z-axis",
 "errorInfo": {}, "wrappedErrors": []}

// 槽 7，弧高 150
{"id": "1a8861f6-5411-4ff7-b850-0393a30ec9a5",
 "createdAt": "2026-09-30T08:40:53.937525Z",
 "isDefined": false,
 "errorType": "FailedToPlanMoveError",
 "errorCode": "4000",
 "detail": "Arc out of bounds in the Z-axis",
 "errorInfo": {}, "wrappedErrors": []}
```

重试由本地脚本的策略完成，就是把弧高降下来再试：`180 -> 150`、`150 -> 130`。因此**记录下来的 180/150/130 是脚本初值与降级步长的产物，不是引擎给出的边界**。

### 待解释的一点

同样是 150 mm：在槽 3 **通过**，在槽 7 **被拒**。按 `FailedToPlanMoveError / 4000 / Arc out of bounds in the Z-axis` 的语义，弧线是否可行取决于起点终点的几何，不只是弧高那个数字。两次运行套用的 labware offset 不同（见下），这可能是一个切入点，但历史记录里没有做过受控对比，**这里不下结论**。

## 三、逐个 run 的已知信息

### `ac20a023-1815-4a08-a9b1-61880ce9cb15`

类型：维护运行。吸头取自槽 2 的 `opentrons_96_tiprack_20ul` A1。

注入的 labware offset（打印出来的几项）：

| 槽 | offset | definitionUri |
|---|---|---|
| 2 | `(+0.20, +0.50, 0.00)` | `opentrons/opentrons_96_tiprack_20ul/1` |
| 3 | `(+0.20, +0.00, −30.00)` | `custom_beta/custom_24_tuberack_eppendorf_2ml_slot3_pitch19p69/1` |
| 7 | `(+2.10, +0.50, +1.00)` | `opentrons/opentrons_24_tuberack_eppendorf_2ml_safelock_snapcap/1` |

命令顺序（脚本自述，非逐条参数）：home → 丢弃已有吸头 → 装载槽 2 吸头盒 → 取 A1 吸头 → 装载目标槽 → 移动。

dry-run 留下的命令级参数形状（槽 3，弧高 180）：

```
moveToAddressableAreaForDropTip {"pipetteId":"<left>","addressableAreaName":"fixedTrash","ignoreTipConfiguration":true,"minimumZHeight":180.0}
dropTipInPlace                 {"pipetteId":"<left>","ignoreTipConfiguration":true}
loadLabware                    {"location":{"slotName":"2"},"loadName":"opentrons_96_tiprack_20ul","namespace":"opentrons","version":1}
pickUpTip                      {"pipetteId":"<left>","labwareId":"<tiprack>","wellName":"A1","wellLocation":{"origin":"top","offset":{"x":0,"y":0,"z":0}}}
loadLabware                    {"location":{"slotName":"3"},"loadName":"custom_24_tuberack_eppendorf_2ml_slot3_pitch19p69","namespace":"custom_beta","version":1}
moveToWell                     {"pipetteId":"<left>","labwareId":"<target>","wellName":"A1","wellLocation":{"origin":"top","offset":{"x":0.0,"y":0.0,"z":0.0}}}
```

对槽 6 的 dry-run 显示 `moveToAddressableAreaForDropTip` 的 `minimumZHeight` 是 **130.0**（首次即 130）。

### `ee3a249b-5d89-4893-b697-32068c63e513`（槽 6）

参数：`--slot 6 --well A1 --override-z 6=-30 --fresh-tip --tip-well B1`

**吸头取自槽 2 吸头盒的 B1**（不是 A1）。注入 offset：槽 2 `(+0.20, +0.50, 0.00)`、槽 6 `(−1.50, +1.00, −30.00)`。

步骤：重置吸头状态（先丢）→ 装载槽 2 吸头盒 → 取 B1 吸头 → 装载槽 6 → 移动到位。

### `bb172223-42f6-401b-90bf-7000dcaea802`（槽 8）

参数：`--slot 8 --well A1 --no-tip`，随后再来一次 `--no-tip --dz -30`

**不丢吸头、不重新装盒、不取新吸头**，复用已挂吸头。注入 offset：槽 2 `(+0.20, +0.50, 0.00)`、槽 8 `(+2.00, −0.30, +1.50)`。

> 另有一个候选值 `(−0.5, 0.0, −28.5)`，只在 dry-run 里打印过，**从未真跑**。

### `1b1a710d-e590-4704-b8a0-377c7fb51e08`（十五点核对）

类型：协议运行。protocolId `38cdda92-f1af-40a4-93f8-945e8df2574e`，创建于 2026-10-01 11:07:38，状态 `paused`（靠协议里的 `waitForResume` 逐点停）。

分析结果：`result ok`、`errors 0`、`commands 65`。

运行统计：启动 **17** 条命令；末尾 8 条的类型为

```
loadPipette, loadPipette, comment, waitForResume, pickUpTip, comment, moveToWell, waitForResume
```

停点 **15** 个（左枪 8、右枪 7）。

注入的 8 个 labware offset：

| 槽 | offset | definitionUri |
|---|---|---|
| 1 | `(−0.3, −5.7, 0.8)` | `opentrons/corning_96_wellplate_360ul_flat/2` |
| 2 | `(0.1, 0.5, 0)` | `opentrons/opentrons_96_tiprack_20ul/1` |
| 3 | `(0, 0, 0)` | `custom_beta/custom_24_tuberack_eppendorf_2ml_slot3_pitch19p69/1` |
| 4 | `(−2, 2, −5.45)` | `opentrons/opentrons_15_tuberack_falcon_15ml_conical/1` |
| 5 | `(0.3, 0, 0)` | `opentrons/opentrons_96_tiprack_300ul/1` |
| 6 / 7 / 8 | `(0, 0, 0)` | `opentrons/opentrons_24_tuberack_eppendorf_2ml_safelock_snapcap/1` |

注意：这里槽 3/6/7/8 是 `(0,0,0)`，而上面 `ac20a023` 用的槽 3 是 `(+0.20,+0.00,−30.00)`。**两次运行用的不是同一版 offset**——`1b1a710d` 用的是 09-30 绝对位置读数那一版，`ac20a023` 用的是 09-29 那一版。这与 `calibration_log.md` 里记的基准漂移是同一件事。

十五个点覆盖：槽 2 / 1 / 3 / 6 / 7 / 8（左枪）与槽 5 / 4-A1 / 4-A2 / 1（右枪）的 A1 附近。

> 停点消息里的中文在导出中损坏成了 `?`，命令序列与坐标不受影响。

## 四、更早的运行（来自标定日志）

| 日期 | run | 事件 |
|---|---|---|
| 2026-09-07 | `06c79436` | 官方协议第一次成功运行 |
| 2026-09-11 | `5ab0f022` | 失败：`ALARM: Hard limit +X` |
| 2026-09-15 | `d8dc418c` | 恢复成功，引入列距修正 |
| 2026-09-22 | `565a5d1c` | 板完成（succeeded） |
| 2026-09-29 | — | 重标定，offset 值陷入混乱（槽 3 `−30`，槽 6/7/8 `+1.5`） |
| 2026-09-30 | — | 定位根因，offset 全部修正为 `(0,0,0)` |

这些 run 同样已不在机器上。
