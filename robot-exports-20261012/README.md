# robot-exports-20261012

这是按照 OT-2 沙箱复现的数据请求整理的一份导出包。内容分两类：机器标定的原始导出，和从历史运行记录里能取到的部分。

## 约定与脱敏

所有文件都做了中性化处理，替换规则如下，**同一实体在所有文件里用同一代号**：

| 原值 | 替换为 |
|---|---|
| 机器名 | 删除，正文里称 "OT-2" |
| 机器人序列号 | `ROBOT-SN` |
| 左枪（P20）序列号 | `PIP-LEFT-P20` |
| 右枪（P300）序列号 | `PIP-RIGHT-P300` |
| 内网地址 | `<ROBOT_IP>` |
| 工作目录绝对路径 | `<WORKDIR>` |
| 自定义耗材定义名前缀 | `custom_24_tuberack_...` |
| 本地脚本名 | 中性名（见下表） |

脚本名对照：

| 中性名 | 作用 |
|---|---|
| `labware_offsets.json` | labware offset 的唯一真源 |
| `offsets.py` / `offsets_check.py` | 加载与校验 |
| `runner.py` | 唯一运行入口，注入 offset、检查基准漂移 |
| `tip_move.py` | 把吸头移到指定孔 |
| `position_capture.py` | 绝对位置捕获 |

## 文件清单

| 文件 | 内容 | 对应请求项 |
|---|---|---|
| `health.json` | `/health` 原始返回：软件版本、固件、系统版本、协议 API 版本范围 | 1 |
| `tip_length.md` | 两个枪的吸头长度摘要 | 2 |
| `custom_labware/custom_24_tuberack_eppendorf_2ml_slot3_pitch19p69.json` | 槽 3 自定义耗材定义，完整文件 | 3 |
| `calibration_at_mark.json` | **标注那一刻**的机器标定快照（六个端点） | 5 |
| `calibration_after_recal.json` | **重标之后**的机器标定导出 | 5 |
| `labware_offsets.json` | labware offset 真源，含 `machine_calibration_at_mark` 与 `protocol_params` | 5 |
| `calibration_log.md` | 标定日志：为什么 offset=0、为什么旧值是错的 | 5 的上下文 |
| `run_records.md` | 能取到的运行记录，以及取不到的部分与原因 | 4 / 6 |

## 为什么有两个标定文件

2026-09-30 这台机器自己被重标过一次（甲板矩阵、左右枪偏移都变了）。标注 offset 时生效的标定，和重标之后生效的标定不是同一套，**两者都在这个包里**：

- `calibration_at_mark.json` —— 标注 offset 当时在用的那套（甲板矩阵 `lastModified` 2026-02-25）
- `calibration_after_recal.json` —— 重标之后的那套（甲板矩阵 `lastModified` 2026-09-30T13:48:12Z）

重放历史运行时用哪一套，取决于那个 run 发生在重标之前还是之后。这个区分对弧高能否复现很可能有影响。

## 没有的东西

- **历史 run 的命令记录取不到**：这台机器的维护运行用完即弃。2026-09-30 探测过 7 个 run，`/runs/{id}` 与 `/runs/{id}/commands` **全部返回 404**。所以请求单第 4 项（`ac20a023` 的完整命令）拿不到，只能给当时落进日志的部分，见 `run_records.md`。
- **09-29 那一版 `labware_offsets.json`**（jog 累加版）：这个包里只有 09-30 版。09-29 的值以对照形式记录在 `calibration_log.md` 里。
- 标准 24 孔架定义、甲板原点表、标定位置数据等：在旧工作目录里，本次未一并带出。
