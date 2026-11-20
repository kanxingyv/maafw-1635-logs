# MaaFramework #1635 日志 / Logs

对应 issue / Issue: https://github.com/MaaXYZ/MaaFramework/issues/1635

- **MaaFramework**: v5.15.0（Built at Nov 2 2026 16:45:58，Windows x64）
- **控制器**: Adb + EmulatorExtras，实际选中 `MuMuPlayerExtras`
- **模拟器**: MuMu Player 12，`127.0.0.1:16448`
- **宿主**: Narutomobile v2.16.4（MFAAvalonia 内核）
- **管线空间**: 1280×720

---

## 文件清单 / Files

| 文件 | 行数 | 说明 |
| --- | --- | --- |
| `maafw-2026-11-19_12-57-09_to_13-38-59_STALL.log` | 70713 | **卡死窗口**，从 `wait_freezes` 启动前一直到手动停止前 |
| `maafw-2026-11-19_13-39-12_to_13-49-15_AFTER_RESTART.log` | 59763 | **对照**：重建 tasker 之后同一任务正常跑完的日志 |
| `host-app-log-20261119.log` | 1656 | 宿主侧日志，含任务队列、`[内存管理]` 心跳、停止重试 |

---

## 时间线 / Timeline

| 时间 | 事件 |
| --- | --- |
| 12:56:46.988 | `_screencap  Image size changed [width=1600] [height=900] [image.cols=900] [image.rows=1600]` |
| 12:56:47.255 | 同上，仍为竖屏 900×1600 |
| 12:57:03.025 | `Image size changed [width=900] [height=1600] [image.cols=1600] [image.rows=900]` |
| **12:57:09.407** | `ActionHelper::wait_freezes  [param.time=300ms] [param.rate_limit=1000ms] [param.timeout=20000ms]`，`Node.WaitFreezes.Starting` |
| 12:57:09.708 | 最后一次比较 `get_liveness_award_box_2_wait_freezes [score=0.137648] [filtered_results_=[]]` |
| **12:57:09.792** | `screencap failed after retries, force re-connect [i=0]` → `adb kill-server` → `adb connect` |
| 12:57:14.914 | 最后一次 `Image size changed`，随后 `calc_target_image_size` 重算为 1280×720 |
| **12:57:15.032 → 13:38:59.998** | 无任何节点事件；`get_display_id` 每秒约 30 次 |
| 13:39:06 | 手动停止，宿主侧 10 次均未拿到有效状态 |
| **13:39:12** | 重建 tasker，立刻恢复；同一任务 4 分 01 秒跑完，`Image size changed` = 0 |

---

## 卡死窗口的构成 / What the stall window contains

```
总行数             : 70713
get_display_id     : 70146   （约 30 次/秒，全部 [display_id=0]）
其他行             :   567   （全部在 12:57:15 之前）
12:57:15 之后       : 70135 行，100% 是 get_display_id，其余 0 行
```

`MuMuPlayerExtras.cpp L570 get_display_id` 是窗口里唯一持续输出的日志，没有 Error、没有 Warn、没有 `re-screencap`、没有节点事件。`Node.WaitFreezes.Starting`（12:57:09.407）之后既没有 `Succeeded` 也没有 `Failed`，`timeout=20000ms` 从未触发。

`AdbControlUnitMgr::screencap()` 的重试是有界的（3×10），窗口里只留下了开头那一条 `force re-connect [i=0]`，之后再没有 `re-screencap` —— 说明卡点不在这一层。

---

## 关于缺失的前段 / Missing segment

宿主的日志清理在 11/20 轮转时删掉了 11/19 的 `maafw.bak.*`，所以 **12:55:40 – 12:57:09** 这一段已经不在磁盘上，它包含三次 `Image size changed`（12:56:46.988 / 12:56:47.255 / 12:57:03.025）。这些行在 issue 正文里保留了当时的原文摘录。

卡死窗口本身（`wait_freezes` 启动 → 手动停止前）是完整的，未做任何裁剪或修改。

---

## 触发场景 / Trigger

横屏游戏（火影忍者）内部有竖屏 H5 页面（"金币助手"），打开时安卓显示面真的转成竖屏，抓帧从 1600×900 变成 900×1600（缩放后 1280×720 → 720×1280）。这与模拟器窗口无关，是应用自身切换了方向。
