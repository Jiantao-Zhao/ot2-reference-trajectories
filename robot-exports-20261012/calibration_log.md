# 标定日志 — 2026-09-30：定位并修复 offset 标定的根本缺陷

Date: 2026-09-30 · Robot: OT-2 (API 26.6.0, fw v1.1.0-25e5cea)

## 摘要

经过反复标定始终无法校准、Z 高度记录混乱的问题，最终定位到**根因是 `do_mark` 的相对累加逻辑，而非 offset 机制本身**。改用**绝对位置测量**后，一次就测出正确答案：四个 24 孔架的 labware offset 都是 **`(0, 0, 0)`**——官方定义 + 机器已有标定本身就精确匹配物理现实，之前那些 `-29/-30/+1.5` 值都是错误叠加。


## 背景：为什么反复 calibrate 都无法校准

时间线：

| 时间 | 事件 |
|---|---|
| 2026-09-07 | 官方协议第一次成功运行（run `06c79436`） |
| 2026-09-11 | 失败：`ALARM: Hard limit +X`（run `5ab0f022`） |
| 2026-09-15 | 恢复成功（run `d8dc418c`），引入列距修正等 |
| 2026-09-22 | 板完成（run `565a5d1c` succeeded） |
| 2026-09-29 | 重标定，但 offset 值陷入混乱：slot3=`-30`，slot6/7/8=`+1.5` |
| 2026-09-30 | **本次**：定位根因，offset 全部修正为 `(0,0,0)` |

09-29 之后，同样的 24 孔架，offset 却互相矛盾（同一个物理架子差 30 mm），反复标定也无法收敛到正确值。

## 根因分析（4 个技术根因，按重要性排序）

### 根因 1（核心）：`do_mark` 是"相对累加"，不是"绝对测量"

`the local control console/server.py` 里的 `do_mark()` 原逻辑：

```python
well_offset = run_offset + jog
#            ↑ 当前运行已生效的旧 offset   ↑ 这次手动 jog 的量
```

它记录的不是"孔真实偏了多少"，而是"**旧 offset + 又微调了多少**"。如果运行里挂着错误的旧值（如 `-29`），手动调 `+30` 想修正，记下的却是 `-29 + 30 = +1`，而不是"真实偏差是 0"。

**于是每次校准都在旧错误值上叠加，永远到不了正确的绝对值**——这就是"反复 calibrate 也校准不了"的直接原因。

### 根因 2：没有任何"绝对参照"来验证

之前的方法从头到尾无法独立回答"这个 offset 对不对"，只能靠"目视 + 相对 jog"（主观 + 累积误差），跑一次撞一次再回来调。

### 根因 3：维护运行是易失的，jog 历史会丢

这台机器的维护运行"用完即弃"——实测所有历史运行全部返回 404。而 `do_mark` 依赖的 `run_offset` 和 jog 历史都存在运行里，运行一被替换数据就静默丢失。

### 根因 4：基准漂移了约 30 mm

旧值 `-29/-30` 和后来的 `+1.5` 之间差 30 mm——同一个物理架子 offset 差 30 mm，说明机器标定基准在不同时间点变过。旧 offset 在新基准下全体作废。

## 这次成功的方法（绝对测量）

核心：**零偏移基准 + 读绝对位置**，一步到位：

1. **零偏移运行**：把所有 labware offset 清成 0 → 指令点 = 模型点（不再有未知平移）
2. **读绝对位置**：`moveRelative(distance=0)` 返回当前绝对坐标且**不动任何轴**（连读三次数值一致，已验证）
3. **算 offset**：`offset = 绝对位置 − 模型位置`

```
槽位 6 原点 [265, 90.5, 0] + A1 孔位 (18.21, 75.43, 41.27) + offset 0
X = 265.0 + 18.21 = 283.21    实测 283.21 ✓
Y = 90.5  + 75.43 = 165.93    实测 165.93 ✓
Z = 0 + 41.27 + 39.1 = 80.37  实测 80.37 ✓
```

四个架子的结果互相锁死（X/Y 各异但 Z 全等 80.37），交叉验证了结论。

## 关键结论

**四个 24 孔架（槽位 3/6/7/8）的 labware offset 都是 `(0, 0, 0)`。**

也就是说：官方 labware 定义 + 机器 2026 年 2~6 月已有的标定，本身就精确匹配物理现实，不需要任何修正。之前写进 `labware_offsets.json` 的 `-29/-30/+1.5` 等值，是**错误叠加**，把本来就对的位置推偏了约 30 mm。

`labware_offsets.json` 存 offset 这个机制（等价于 Opentrons 标准的 labware offset）**从来是对的**；坏掉的是"测出正确 offset 并写进去"的方法。

## 代码改动（`the local control console/server.py`）

| 改动 | 作用 |
|---|---|
| STATE 加 `model_position` 字段 | 存 move_well 时读到的绝对模型位置 |
| 新增 `_read_position(rid, pid)` | 用 `moveRelative(distance=0)` 读绝对位置 |
| agent 版 + HTTP 版 move_well | move 成功后读绝对位置存 `model_position` |
| **`do_mark` 核心重写** | 从 `run_offset + jog` 改为 `绝对位置 − 模型位置` |
| 非零偏移时打 WARNING | 防止在 normal 模式下静默写错 |

新流程：**move_well（记录模型位置）→ 手动 jog → mark（读绝对位置减模型位置）**，完全不依赖 jog 累加、不依赖 `run_offset`、不依赖易失的运行历史。

### 前提：标定必须零偏移模式

绝对测量只有在零偏移基准下才是"绝对 offset"。启动服务时需 `CALIB_ZERO_BASE=1`（代码里 `_build_offsets()` 返回空）。服务已改为在 **Windows** 上直接运行（不依赖 WSL），`server.py` 只依赖标准库 + 跨平台相对路径。

## 复用的数据文件

见 `calibration_position_data/` 文件夹（本次标定依据的全部位置数据 + 结果）：

| 文件 | 内容 |
|---|---|
| `ot2_standard_positions.json` | 甲板定义：12 槽位原点、标定点 |
| `opentrons_24_tuberack_eppendorf_2ml_safelock_snapcap.json` | 标准 24 孔架定义（槽位 6/7/8） |
| `custom_24_tuberack_eppendorf_2ml_slot3_pitch19p69.json` | 槽位 3 自定义定义（列距 19.69） |
| `opentrons-OT-2-calibration.json` | 机器标定（deck 矩阵 + pipette + tip length） |
| `calibrated_deck_positions.json` | 最终坐标结果（机器可直接读） |
| `README.md` | 说明 + 复用条件 |

## 复用条件和限制

这套 offset=`(0,0,0)` 的结论，**只在以下机器标定保持当前值时有效**：

| 项 | 值 | lastModified |
|---|---|---|
| 甲板标定矩阵 | `[[0.9992,0.004,0],[0,1.0016,0],[0,0,1]]` | 2026-02-25 |
| 左枪 pipette offset | `[0.2, -5.7, 0.1]` | 2026-06-02 |
| P20 吸头长度 | `30.85 mm` | 2026-02-25 |

**任何一项被重新标定，坐标和 offset 都要重新测量**（尤其甲板矩阵——它定义"甲板坐标↔马达坐标"的换算）。

## 以后怎么检查 / 复核

随时可用这个绝对判定独立验证（不依赖任何历史记录）：

```
零偏移运行 → 吸头移到某架的模型孔顶 → 目视确认是否在管口中心
```

若四个架子都能在 `dz=0` 落于管口，说明 offset=`0` 仍成立。

标定新 labware 的标准流程（绝对测量）：

```
1. CALIB_ZERO_BASE=1 启动 UI
2. move_well 到该 labware 的模型孔顶（服务记录 model_position）
3. 手动 jog 到管口中心
4. mark（读绝对位置减模型位置，直接写 labware_offsets.json）
5. 不碰 do_mark 的 jog 累加，不叠旧值
```

## 其他发现

- **`Z 行程上限 = 172.15 mm`**（从串口日志 `M365.0 Z172.15` 读到）。手动移动的 `minimumZHeight` 必须低于它，否则引擎报 `Arc out of bounds in the Z-axis`。
- 这台 API（26.6.0）没有实时位置接口（`/robot/state`、`/robot/settings` 等全部 404），唯一可靠的绝对位置来源是 `moveRelative(distance=0)` 的**当次返回**（`/commands` 里存的 `result` 是 `null`）。
- `transfer_20260907_singles95_no_aldehyde_OT2_recovery_A7-H12.py` 引用了未定义的 `ASPIRATE_ABOVE_BOTTOM_MM`（第 434 行），该 recovery 协议已无法运行（详见 `audit_20260929_offsets.md`）。

## 遗留问题

- **槽位 4（Falcon，`z=+25.5`）未复核**：它还带着 09-29 那批 `+30` 改动残留，且用的是 P300（offset 和吸头长度是另一套）。它影响全板 buffer/hydrazide 的加液精度，建议下次跑协议前用同样方法重测。
- 槽位 1/2/5 的 X/Y 小偏移未在新基准下复验（因基准变化主要体现在 Z，X/Y 分量差异很小，判定可保留，但未实测确认）。

---
