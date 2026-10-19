# Microduck 动作包：节奏舞（点头 + 摇头）

给 [pollen-robotics/microduck_rl](https://github.com/pollen-robotics/microduck_rl) 仿真使用的**命令级动作包**。

不训练新策略、不直接驱动关节——只通过 `head_pose`（头部四轴）和 `body_pose`（躯干六轴）两个**命令通道**编排动作，平衡由策略自行负责。

## 效果预览

![节奏舞 · 摇头晃脑](media/demo.webp)

*15 秒实拍循环（2 倍速播放）。完整说明见下文。*

---

## 1. 文件清单（每个文件干什么用）

| 文件 | 作用 | 是否必需 |
|---|---|---|
| `actions/nod_and_shake.json` | 动作序列本体：27 步「节奏舞」（点头 ±0.6 rad + 左右甩头 ±1.2 rad，三段完整摆动，首尾闭合） | ✅ 必需 |
| `squat.onnx` | **支持躯干起伏（下蹲/伸展）通道的站立策略**——为什么需要它，见第 2 节 | 想用 `body.z` 时必需 |
| `infer_policy_patched.py` | 原版 `scripts/infer_policy.py` + 动作序列播放支持（`--sequence` 系列参数）。直接替换仓库里的同名脚本即可 | ✅ 必需 |
| `README.md` | 本文（中文完整说明） | — |
| `README_EN.md` | English summary (same content, shorter) | — |
| `LICENSE` | Apache License 2.0（覆盖代码，来自上游仓库） | — |
| `LICENSE-MODELS.md` | 3D 资产许可 CC BY-NC-SA 4.0（来自上游仓库） | — |

---

## 2. 为什么会有 `squat.onnx`（正常情况下是不需要的）

### 短答案

- **只想做头部动作**（点头/摇头/歪头）→ 用官方 stand / walking 策略就行，**不需要** `squat.onnx`；
- **想让 `body_pose` 的 z 命令（下蹲 / 伸展）真的生效** → 才需要它。

### 长答案（这里是个坑，值得读）

microduck 的策略是 **command-conditioned** 的：观测里带 13 维命令

```
[twist 3] + [head_pose 4] + [body_pose 6]
```

但官方 StandUp / Velocity 任务对 **`body_pose` 通道采用了"延迟开启"的课程**：

| 训练轮数 | body_pose_tracking 权重 | z 命令范围 |
|---|---|---|
| 0 – 2499 | **0（完全关闭）** | ±5 mm |
| 2500 | 1.5 | −10 / +5 mm |
| 3000 | 3.0 | −18 / +8 mm |
| 4000 | **4.0** | **−40 / +30 mm** |

也就是说：**如果你的 StandUp 策略只训练了 1000–2000 轮，`body_pose` 通道根本没被训练过**——在 `infer_policy.py` 里按 `B` 进体态模式、按上下键调 z，机器人**毫无反应**（实测：命令 −40 mm，实际只动 1.8 mm）。

本包里的 `squat.onnx` 就是为这个场景专门训练的：

- 基于 `Mjlab-Squat-Flat-MicroDuck` 任务（本文作者自建）；
- 把 body z 的命令课程从 2500 轮压缩到 **300 轮内拉满**；
- 删除 locomotion 奖励（允许双脚不动）、用 z-only 的高度跟踪奖励；
- 训练 800 轮，实测 **命令 −40 mm → 实际 −40.0 mm**（0 次摔倒）。

所以它在本包里充当 **`--standing` 槽位的策略**，替代普通的 StandUp。

> 一句话：**`squat.onnx` 不是"另一个走路策略"，它是"唯一能让下蹲命令生效的站立策略"。** 不需要蹲起动作的话，可以删掉它，改用官方 StandUp。

---

## 3. 快速开始

```bash
# 1) 准备环境（原仓库依赖）
cd microduck_rl
uv sync

# 2) 放入本包文件
cp action_share/squat.onnx .
cp action_share/infer_policy_patched.py scripts/infer_policy.py   # 覆盖前请先备份原文件
cp -r action_share/actions .

# 3) 运行（--walking / --roulade 是可选策略，没有可以省略）
uv run scripts/infer_policy.py \
    --standing squat.onnx \
    --walking output.onnx \
    --roulade roulade.onnx \
    --new-cmd-obs \
    --sequence actions/nod_and_shake.json
```

启动后终端会提示：

```
Sequence: 节奏舞（长版 …）—— 27 steps × 0.25s (speed ×2, loop=on …)
press '1': start looping  /  press again: stop
```

- 按 **`1`** → 开始循环播放（默认 2 倍速，约 6.8 秒一轮）
- 再按 **`1`** → **立即停止**，命令归零、控制权交还键盘
- `Esc` 或 `Q` 退出

可用参数：

| 参数 | 默认 | 说明 |
|---|---|---|
| `--sequence <file>` | — | 动作序列 JSON |
| `--sequence-speed <x>` | 2.0 | 倍速（step_dt 除以该值） |
| `--sequence-loop` | 按键模式自动开 | 循环播放 |
| `--sequence-key <char>` | `1` | 开关按键 |
| `--sequence-auto` | 关 | 启动即播（不等按键） |

---

## 4. 动作序列格式（自己写新动作）

```json
{
  "name": "我的动作",
  "step_dt": 0.5,
  "steps": [
    {"head": {"yaw": 0.4, "pitch": 0.3}},
    {"head": {"yaw": -0.4}, "body": {"z": -0.03}},
    {"head": {"yaw": 0.0, "pitch": -0.3}}
  ]
}
```

规则：

1. **所有数值都是相对上一步的增量**（不是绝对角）；
2. 一个 `step` 里可以写多个通道，**同时生效**；
3. **每轮必须闭合**：head 四轴、body 六轴的累计值都要回到 0，否则动作结束会停在歪斜姿态；
4. 单位：角度用 **rad**，位置用 **m**（30 mm 写 `0.030`）；
5. 播放时按 `step_dt` 做 **smoothstep 插值**，不会阶跃跳变。

可在 `head` 里用的键：`neck` / `pitch` / `yaw` / `roll`
可在 `body` 里用的键：`x` / `y` / `z` / `roll` / `pitch` / `yaw`

### 实测安全边界（squat 策略，800 轮）

| 通道 | 安全命令范围 | 实测备注 |
|---|---|---|
| `head.pitch` | **±0.80** | ±1.2 会摔（150 步内 10 次）；+1.222 是机械上限 |
| `head.yaw` | **±1.20** | 1.2 仍 0 摔倒，但跟随率降到 66%（实际到头约 45°） |
| `head.roll` | **±0.30** | 机械限位 ±0.4363（±25°） |
| `head.neck` | **−1.9 ~ +0.65** | 机械上限 +0.698，别顶死 |
| `body.z` | **−0.040 ~ +0.030 m** | −40 mm 蹲满（实测到位） |
| `body.x` / `y` | ±0.005 m | 常驻噪声范围 |
| `body.roll/pitch/yaw` | ±0.26 rad | 15° |

> 这些是**策略能力边界**，不是机械边界——超了不会立刻坏，但动作会不到位或失稳。

---

## 5. `infer_policy_patched.py` 改了什么

相对原版 `scripts/infer_policy.py` 的全部改动（都围绕序列播放）：

1. 新增 argparse 参数：`--sequence`、`--sequence-speed`、`--sequence-loop`、`--sequence-key`、`--sequence-auto`；
2. `main()` 中加载 JSON，把**相对增量预累加**成绝对命令序列；
3. 主循环中按时间取步、smoothstep 插值，写入 `policy.head_offset` / `policy.body_cmd` 并调用 `policy._update_command()`；
4. 按键分支：`1`（可配置）切换播放/停止，停止时立即把命令归零；
5. 循环模式下对时间取模；一次性模式结束后自动交还控制权。

推理、观测构建、BAM 执行器、延迟/噪声等全部**未改动**。

---

## 6. 许可与致谢

- **代码**（`infer_policy_patched.py` 及原仓库）：[Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0)，来自 pollen-robotics/microduck_rl，保留原版权声明；
- **3D 模型 / MJCF 资产**：CC BY-NC-SA 4.0（非商业、署名、相同方式共享），见原仓库 `LICENSE-MODELS.md`；
- **`squat.onnx`**：基于上述代码与资产训练得到的派生模型，随包提供，**仅供非商业研究使用**；
- **`actions/*.json`**：动作编排数据，可自由使用与修改。

---

## 7. 已知限制

1. **跟随率不是 100%**：命令越大越跟不上（yaw 1.2 → 实际 0.79 rad；pitch 0.8 → 实际 0.57）；
2. **高速播放会"糊"**：2 倍速下每步 0.25 s，大幅动作超出策略响应能力，视觉上会变成小幅快抖；
3. **头部动作不需要 squat.onnx，但躯干 z 需要**——见第 2 节；
4. 真机部署需固件支持 13D 命令布局，且策略需按部署流程导出（obs 归一化器已烘焙进 ONNX）。
