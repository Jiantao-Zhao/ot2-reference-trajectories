# 吸头长度标定

来自机器 `/calibration/tip_length`。**吸头长度按移液器序列号对应**，代号与 `README.md` 的约定一致。

## 重标之后（`calibration_after_recal.json`，2026-09-30）

| 移液器 | 代号 | 吸头长度 (mm) | 对应吸头盒 | lastModified |
|---|---|---|---|---|
| 左（P20） | `PIP-LEFT-P20` | **30.85** | `opentrons/opentrons_96_tiprack_20ul/1` | 2026-02-25T09:35:13Z |
| 右（P300） | `PIP-RIGHT-P300` | **48.70** | `opentrons/opentrons_96_tiprack_300ul/1` | 2026-06-15T14:24:14Z |

## 标注 offset 时（`calibration_at_mark.json`，2026-09-30 11:18 快照）

两个枪的吸头长度与上表**相同**：左 30.85（`lastModified` 2026-02-25T09:35:13Z）、右 48.70（2026-06-15T14:24:14Z）。

即：重标改变的是甲板矩阵与移液器偏移，**吸头长度没有变**。

## 对应的吸头盒指纹

| 吸头盒 | tiprack 指纹（definition 内容摘要） |
|---|---|
| `opentrons_96_tiprack_20ul/1` | `60e107ba7e6e5d23beefe9f0c888f860f7b923c71906aaa5e7f460ab25c4f38a` |
| `opentrons_96_tiprack_300ul/1` | `ed323db6ca1ddf197aeb20667c1a7a91c89cfb2f931f45079d483928da056812` |

这两个指纹在 `calibration_at_mark.json` 与 `calibration_after_recal.json` 里都出现，可用来确认引用的是同一支吸头盒。

## 备注

按请求单的推理，弧高上限约等于「移液器最高位置 − 吸头有效长度 − 1 mm」。这里两个枪的吸头长度差 17.85 mm，如果弧高上限确实随吸头长度平移，那么用左枪（P20）和右枪（P300）做同高度的弧线，判定结果会不一样——这一点在历史记录里没有被单独测过。
