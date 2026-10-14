# 五次弧高结果的复现

## 结论

`run_records.md` 里的五次弧高结果，在官方规划器里全部能复现（5/5）。同样是 150 mm、槽 3 通过、槽 7 被拒，原因是**引擎当时是否认为挂着吸头**，不是起点终点的几何，也不是两次套用的 labware offset 不同。

## 依据

- 五次都是 `moveToAddressableAreaForDropTip`（去垃圾桶丢吸头），并且每次调用的顺序都是先丢、再取吸头（`run_records.md` 第三节的 dry-run 输出）。
- 槽 3 和槽 7 用的是同一个维护运行 `ac20a023`：槽 3 那次是这个运行里的第一次调用，引擎里还没有吸头；槽 7 那次接在后面，槽 3 取的吸头还挂着。
- 槽 6 用的是新的运行 `ee3a249b`，开始时引擎里没有吸头。

## 规划器怎么判

官方规划器在 `minimumZHeight` 加 1 mm 超过移液器能到的最高点时，报 `FailedToPlanMoveError`、`4000`、`Arc out of bounds in the Z-axis`。移液器能到的最高点，没挂吸头时是 180.6 mm，挂着吸头时要再减去吸头的有效长度。

| 情况 | 能接受的最高 `minimumZHeight` |
| --- | ---: |
| 没挂吸头 | 179.59 mm |
| 挂 20 µL 吸头，按耗材定义的长度（30.95 mm） | 148.65 mm |
| 挂 20 µL 吸头，按机器标定的长度（30.85 mm） | 148.74 mm |

`minimumZHeight` 高于所有耗材时，这个判定只看这两个数。按默认标定，槽 3、6、7、8 的上限完全相同。

## 复现结果

| run | 槽 | 弧高 | 引擎里有没有吸头 | 记录 | 规划器 |
| --- | --- | ---: | --- | --- | --- |
| ac20a023 | 3 | 180 mm | 没有 | 被拒 | 被拒 |
| ac20a023 | 3 | 150 mm | 没有 | 通过 | 通过 |
| ac20a023 | 7 | 150 mm | 有 | 被拒 | 被拒 |
| ac20a023 | 7 | 130 mm | 有 | 通过 | 通过 |
| ee3a249b | 6 | 130 mm | 没有 | 通过 | 通过 |

按定义长度和按标定长度，结果都是 5/5。

## 怎么跑

```bash
pip install "opentrons==8.8.2"
python tools/replay_arc_records.py
python tools/replay_arc_records.py --tip-length-mm 30.85
```

`protocol_api` 的 `drop_tip()` 不接受 `minimum_z_height`，所以脚本像当时的主机侧脚本一样，直接把丢吸头的移动发给引擎。这是复现记录用的工具，不是协议，不受操作规范对协议的约束。

## 版本

机器是 26.6.0，属于 opentrons-ot2 版本线，不在 PyPI 上。PyPI 上的 `opentrons` 从 9.x 起，`opentrons analyze` 和 `opentrons simulate` 都会拒绝 OT-2 的 protocol；8.8.2 是 PyPI 上还接受 OT-2 的最新版本，上面的结果都是用它跑的，没有在 26.6.0 上核对。

## 对轨迹一的含义

- 被拒的是主机侧脚本给丢吸头指定的 180 和 150。用 `protocol_api` 写的 `drop_tip()` 没有这个参数。
- 挂着 20 µL 吸头时，`move_to` 的 `minimum_z_height` 不超过 148.7 mm 就会被接受，和目标在哪个槽无关。
- 新开的维护运行如果不知道物理上已经挂着吸头，规划就按裸枪头算，吸头末端实际会比规划低一个吸头长度（P20 约 31 mm）。这是从规划器的判定推出来的，没有在真机上核对过。
