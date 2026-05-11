# §7.5 新鲜度实验手册（最低可执行标准）

本手册对应论文 §7.5（Parameter Freshness for Time-Varying Materials），覆盖四层证据 A/B/C/D。手册的目标是让一个没有读过论文的人也能完整执行 lab 部分，并产出可直接进入论文 §7.5 的图表。

> **铁律**：第 9 节的 pre-registered 判据**在采任何一段视频之前**就必须钉死。一旦开始采数据，判据不可调整。

---

## 0. 总览

| Layer | 性质 | 是否需要实验台 | 时间预算 |
|---|---|---|---|
| A | 合成（identifiability 下限） | 否，纯脚本 | 1 天计算 |
| B | 合成（sim-in-loop staleness 反事实） | 否，纯脚本 | 0.5–1 天计算 |
| C | 真实材料 within-batch | **是**，主体 lab 工作 | 5–7 小时 lab + 1 天分析 |
| D | 真实材料 staleness 验证 | **是**，1 个额外 session | 1 小时 lab + 0.5 天分析 |

**Layer C 的 lab 产物**：54 段 dam-break 视频
**Layer D 的 lab 产物**：额外 2 段 dam-break 视频（same-batch delayed）

---

## 1. 实验范围与四层证据

### 1.1 中心 claim
Hamamichi two-setup 协议在 sim-in-loop 实现下，setup-1 → setup-2 之间有 ~8 h 结构性间隔。本节证明：

- **A**：3 点 cadence ${0, 20, 40}$ min 能识别的 $T_k$ 范围
- **B**：8 h 间隔在合成材料上引入多大 bias
- **C**：6+2 个自制材料的 $T_k$ 落在 A 的可识别窗内
- **D**：真实材料 staleness bias 与 B 的合成预测方向/量级一致

### 1.2 哪些 condition 与哪些 layer 对应

| Condition | Conc. | Texture | 24h rheo? | 涉及 layer |
|---|---|---|---|---|
| Instant oats | 5% | chunky | ✗ | C |
| Instant oats | 9% | chunky | ✗ | C（**双 batch**）|
| Katakuriko starch | 3% | smooth | ✓ | C |
| Katakuriko starch | 5% | smooth | ✓ | C（**双 batch**），D.1, D.2 |
| Katakuriko starch | 7% | smooth | ✓ | C |
| Shiratama flour | 7% | smooth | ✓ | C |
| Pancake mix | 70% | chunky | ✗ | C |

---

## 2. 设备与材料清单

### 2.1 一次性消耗品（按整个实验周期估算）

| 项目 | 用量 | 备注 |
|---|---|---|
| 即食燕麦（Instant oats）| 100 g | oats 5%（20 g）+ oats 9% ×2 batch（72 g × 2 = 64 g）|
| 片栗粉（Katakuriko, potato starch）| 200 g | 3% (12g) + 5% ×2 batch (40g) + 7% (28g) ≈ 80 g, 加 D.1 备用 |
| 白玉粉（Shiratama, sweet rice flour）| 50 g | 7%（28 g）|
| パンケーキミックス（Pancake mix）| 400 g | 70%（280 g） |
| 蒸馏水 / RO 水 | 5 L | 总量 ≈ 4 L；buffer 1 L |
| 一次性手套 | 1 盒 | 防止材料污染 |
| 一次性容器（混合用 400 g 批次）| 15 个 | 9 sessions + 1 Layer D + 5 备用 |

> **关键**：所有粉末从同一袋取，避免批间变化进入"同条件差异"。
> 所有水从同一桶取，温度 25 °C ± 0.5 °C。

### 2.2 不消耗器材

| 项目 | 数量 | 备注 |
|---|---|---|
| 溃坝实验装置（acrylic 三面板，可调 W、H）| 1 套 | 沿用 Hamamichi 2023 设计 |
| 印刷 ChArUco 板 | 1 块 | 沿用 §5 标定流程 |
| 智能手机（支持 240 fps slo-mo）| 1 部 | iPhone 11+ 或 Galaxy S10+ |
| 三脚架 | 1 个 | 必须固定位置不变 |
| 均匀背景板（与材料反差色）| 1 块 | 例如黑底 vs 白色淀粉 |
| 补光灯（≥ 5000 lux）| 1 套 | 避免阴影使 silhouette 不清 |
| 电子秤（精度 0.1 g）| 1 台 | |
| 量杯 / 移液管（精度 1 mL）| 1 套 | |
| 不锈钢搅拌钵 + 打蛋器 | 1 套 | 标准化搅拌时间见 §6 |
| 温度计（精度 0.1 °C）| 1 支 | 校准室温与材料温 |
| 水浴或恒温箱（25 °C）| 1 台 | 控温 ±0.5 °C |
| 秒表 / 计时器 | 1 个 | 三个时点的对时基准 |
| 笔记本电脑（运行 inverse pipeline）| 1 台 | session 中要现场跑 inverse 提议 setup-2 |
| 数据备份硬盘 / 云盘 | 1 个 | session 结束 immediate backup |

### 2.3 标准容器规格

容器内壁尺寸记号为 $(W, H)$，单位 cm。可用网格：
- W: 2.0 / 2.5 / 3.0 / 3.5 / 4.0 / 4.5 / 5.0 / 5.5 / 6.0 / 6.5 / 7.0 cm
- H: 2.0 / 2.5 / 3.0 / 3.5 / 4.0 / 4.5 / 5.0 / 5.5 / 6.0 / 6.5 / 7.0 cm

最少备齐 4 套（不同 W、H 组合），允许 setup-1 在固定 $(W_1, H_1)$、setup-2 在 proposer 提议的 $(W_2, H_2)$ 时不冲突。

---

## 3. 实验前校准（每会话开始时执行）

每个新 session（即每次开新材料 batch）必须完整跑一遍这个 checklist。

```
[ ] 1. 室温 + 水温 25.0 ± 0.5 °C，记录两个数值
[ ] 2. ChArUco 板放回标定位置，与溃坝装置相对位置不变
[ ] 3. 拍背景帧 I_bg（无材料、灯光开、ChArUco 在场）
[ ] 4. 跑 §5 Stage 1 ChArUco 标定，确认 reprojection 误差 < 2 px
[ ] 5. 测试拍：往任一容器倒 50 mL 水，触发释放，确认：
      - 240 fps 模式打开
      - 帧无运动模糊
      - 释放瞬间能从视频上明确判定
[ ] 6. 删测试视频
[ ] 7. 在 lab notebook 写 session header：日期、时间、室温/水温、操作者
```

**任何一项 fail 就 abort session**——data quality 比时间表更重要。

---

## 4. 数据命名与目录结构

### 4.1 目录结构

```
experiments/
  raw/
    YYYYMMDD_<condition>_b<batch>/
      session_log.md
      I_bg.jpg
      charuco_calib.json
      t000_setup1.mp4
      t000_setup2.mp4
      t020_setup1.mp4
      t020_setup2.mp4
      t040_setup1.mp4
      t040_setup2.mp4
      proposed_geometries.json     # 三个时点 setup-2 的 (W,H)
      env_log.csv                  # 每时点的温度、湿度
```

`<condition>` 取值：`oats05` / `oats09` / `kata03` / `kata05` / `kata07` / `shira07` / `pmix70`

`<batch>` 取值：单 batch 条件用 `a`；双 batch 条件（oats09, kata05）用 `a` 和 `b`。

### 4.2 文件名约定示例

- `20260612_kata05_a/t000_setup1.mp4`：2026 年 6 月 12 日，katakuriko 5%，batch A，时点 t=0，setup-1
- `20260612_kata05_a/t020_setup2.mp4`：同 session，时点 t=20 min，setup-2

### 4.3 Session log 模板

每个 session 一份 `session_log.md`，必须包含：

```markdown
# Session: <YYYYMMDD>_<condition>_b<batch>

## 环境
- 室温: 25.X °C
- 水温: 25.X °C
- 湿度: XX %RH
- 操作者: [姓名首字母]

## 校准
- ChArUco reprojection error (Stage 1): _.XX px
- I_bg: I_bg.jpg

## 材料
- 配方实际值（见 §6.2 表）
  - 粉末: ____ g （目标 ____ g）
  - 水: ____ g （目标 ____ g）
  - 实际 mass fraction: ____%
- 搅拌完成时间（绝对时刻）: HH:MM:SS  ← 这是 t=0 的 reference

## 三个时点
| 时点 | 实际释放时刻 setup-1 | 实际释放时刻 setup-2 | 提议 (W2, H2) | 异常 |
|---|---|---|---|---|
| t=0 | HH:MM:SS | HH:MM:SS | (X.X, X.X) | |
| t=20 | HH:MM:SS | HH:MM:SS | (X.X, X.X) | |
| t=40 | HH:MM:SS | HH:MM:SS | (X.X, X.X) | |

## 备份
- [ ] 全部 6 个 mp4 拷贝到云盘
- [ ] session_log.md 拷贝到 git repo
```

---

## 5. 通用 within-batch session 流程

### 5.1 时间轴示意（单 condition）

```
T 之前：校准（§3） + 准备粉末与水（§6.2）
T = 搅拌完成瞬间（记为 t=0 reference，秒表清零）

t = 0 min 时点：
  t=0..1   倒入 setup-1 容器 → 释放 → 录视频
  t=1..3   现场跑 inverse on setup-1 video → 得 θ̂_1
           → proposer 输出 (W2, H2)
  t=3..4   倒入 setup-2 容器 → 释放 → 录视频
  t=4..18  等待 / 把材料保持在 25 °C 水浴

t = 20 min 时点（从 reference 计起）：
  t=20..21  倒入 setup-1 容器（固定 (W1,H1)！）→ 释放 → 录视频
  t=21..23  现场跑 inverse on setup-1@20 → 得 θ̂_1(20)
            → proposer 输出 (W2', H2')  ← 注意此次 setup-2 几何
                                          由 t=20 的 inverse 重提
  t=23..24  倒入 setup-2 容器 → 释放 → 录视频
  t=24..38  等待

t = 40 min 时点：
  t=40..41  setup-1 capture（同 (W1,H1)）
  t=41..43  inverse + proposer → (W2'', H2'')
  t=43..44  setup-2 capture
```

> **关键**：setup-1 几何 $(W_1, H_1)$ **整个 session 固定不变**（防止 routing confound）。setup-2 几何在每个时点重新由 proposer 提议。

### 5.2 单时点操作 checklist

每个时点（t=0、t=20、t=40）都必须按这个清单走：

```
[ ] 1. 时点开始：秒表读数已到目标，确认无误
[ ] 2. 从水浴取材料 → 倒入 setup-1 容器至高度 H1（用尺子刻度）
[ ] 3. 触发录像（240 fps），开始 0.5 秒后释放前面板
[ ] 4. 流动停止后停止录像，拷贝到笔记本 inbox
[ ] 5. 跑 inverse on setup-1 video（命令模板见 §8）
[ ] 6. 等 inverse 输出 θ̂_1（约 12 s），proposer 输出 (W_2, H_2)
[ ] 7. 把 (W_2, H_2) 写进 session_log.md
[ ] 8. 倒入 setup-2 容器（高度 H_2）
[ ] 9. 录像 setup-2，同 step 3
[ ] 10. 文件移到 raw/YYYYMMDD_<condition>_b<batch>/，命名 §4.2
[ ] 11. 等到下一时点（用秒表）
```

### 5.3 单条件总时长预算

| 阶段 | 时长 |
|---|---|
| 校准 + 准备 | 15 min |
| t=0 时点（含 inverse + setup-2 propose）| 4 min |
| 等待 t=0 → t=20 | 16 min |
| t=20 时点 | 4 min |
| 等待 t=20 → t=40 | 16 min |
| t=40 时点 | 4 min |
| 备份 + log 收尾 | 10 min |
| **单 session 合计** | **~70 min** |

### 5.4 整个 Layer C 所需 sessions

| Condition | Sessions |
|---|---|
| oats 5% | 1 (a) |
| oats 9% | 2 (a, b) |
| kata 3% | 1 (a) |
| kata 5% | 2 (a, b) |
| kata 7% | 1 (a) |
| shira 7% | 1 (a) |
| pmix 70% | 1 (a) |
| **小计** | **9 sessions** |

加 Layer D.1 same-batch delayed：+1 session（在 §7.1）。

**总 lab 时长** ≈ 10 × 70 min ≈ **11–12 小时**，分 2–3 天做完比较稳。

---

## 6. 各 condition 配方与制备

### 6.1 通用规则

- 单 batch 目标质量 **400 g**（粉 + 水）
- 水温 25 °C（提前从水浴取）
- 搅拌时间 / 方式标准化（见各 condition 子节）
- 搅拌完成的**最后一滴水加入瞬间** = t=0 reference

### 6.2 配方表

| Condition | 粉末 (g) | 水 (g) | Mass frac. | 搅拌 |
|---|---|---|---|---|
| oats 5% | 20.0 | 380.0 | 5.0% | 手搅 30 s，静置 2 min 后再搅 10 s |
| oats 9% | 36.0 | 364.0 | 9.0% | 同上 |
| kata 3% | 12.0 | 388.0 | 3.0% | 打蛋器中速 60 s 至无颗粒 |
| kata 5% | 20.0 | 380.0 | 5.0% | 同上 |
| kata 7% | 28.0 | 372.0 | 7.0% | 同上，可能需 90 s |
| shira 7% | 28.0 | 372.0 | 7.0% | 打蛋器中速 90 s（白玉粉较粘）|
| pmix 70% | 280.0 | 120.0 | 70.0% | 手搅 30 s，会很稠 — 这是预期的 |

> **每个 condition** 实际称取的粉末与水都要在 session_log 里**精确到 0.1 g** 记录。

### 6.3 搅拌方式标准化

- "手搅"：硅胶刮刀，单向画圈 30 s
- "打蛋器中速"：电动打蛋器档位 2/3，垂直插入，画 ∞ 字 60 s
- 搅拌后**不要**有空气泡为主导外观——pmix 70% 例外（其本来就含气泡）

### 6.4 双 batch 的执行顺序

oats 9% 与 katakuriko 5% 各做两次 batch。两次 batch **绝对不能在同一天连着做**——必须分开两天，否则不能算独立 batch。

推荐：
- Day 1：oats 9% (batch a), kata 5% (batch a), kata 3%, shira 7%
- Day 2：oats 9% (batch b), kata 5% (batch b), kata 7%, pmix 70%, oats 5%
- Day 3 (推荐)：Layer D.1 same-batch delayed (kata 5% 第三 batch)

---

## 7. Layer D 实验

### 7.1 D.1 — same-batch delayed setup-2

**目的**：在同一 batch 上测 setup-2 延迟 40 min 引入的 bias。这是 Layer B 在真实材料上的最干净对照。

**执行**（1 个额外 session）：
- Condition：katakuriko 5%（与 Layer C 同 batch 来源，便于比较）
- 流程改成：
  - 搅拌完成 → t=0 释放 setup-1，录像
  - **不**录 t=0 setup-2
  - 等到 t=40 min，**才**释放 setup-2，录像
  - 不录 t=20 任何东西
- 命名：`YYYYMMDD_kata05_D1delayed/`，含 `t000_setup1.mp4` + `t040_setup2.mp4` + session_log

**与 Layer C 对照**：Layer C 已有的 katakuriko 5% batch a 的 t=0 setup-1 + t=0 setup-2 (joint inverse → θ̂_ref)。Layer D.1 的 t=0 setup-1 + t=40 setup-2 (joint inverse → θ̂_delayed)。差异 Δ_real 与 Layer B 在对应 $T_k$ 的合成预测 Δ_synth 对比方向 + 量级。

### 7.2 D.2 — cross-batch repurposing

**目的**：用现有数据复用，不需要新实验。

**执行**：纯分析步骤：
- 把 katakuriko 5% batch a 的 t=0 setup-1 与 batch b 的 t=0 setup-2 配对 → joint inverse → θ̂_cross
- 与 batch a 的 in-batch joint 对比

oats 9% 双 batch 也可以做同样的配对。

---

## 8. 后处理 / 跑代码

### 8.1 Silhouette 提取

按 §5 现有流程：HSV 阈值 + 形态学开闭 + 最大联通分量。脚本入口：

```bash
python scripts/extract_silhouette.py \
  --video raw/<dir>/<file>.mp4 \
  --calib raw/<dir>/charuco_calib.json \
  --frames 8 \
  --out raw/<dir>/<file>_y.json
```

输出：8 帧 flow-front 距离 $\mathbf{y} \in \mathbb{R}^{8}$。

### 8.2 Inverse pipeline（每时点 session 现场跑）

```bash
python scripts/inverse.py \
  --y raw/<dir>/<file>_y.json \
  --W <W1> --H <H1> \
  --profile real_robust \
  --out raw/<dir>/<file>_theta.json
```

输出：$\hat{\boldsymbol{\theta}}$ + Laplace 95% CI。约 12 s。

### 8.3 Proposer

```bash
python scripts/propose_setup2.py \
  --theta raw/<dir>/<file>_theta.json \
  --W1 <W1> --H1 <H1> \
  --out raw/<dir>/proposed_W2_H2.json
```

输出：$(W_2, H_2)$ 到 1 mm 精度。**写进 session_log，并在物理容器上选最近规格**。

### 8.4 Joint two-setup inverse（session 结束后）

```bash
python scripts/inverse_joint.py \
  --y1 t<tt>_setup1_y.json --W1 <W1> --H1 <H1> \
  --y2 t<tt>_setup2_y.json --W2 <W2> --H2 <H2> \
  --profile real_robust \
  --out t<tt>_joint.json
```

每 condition × 每时点跑一次。Output 进入 Avrami 拟合（§8.5）。

### 8.5 Avrami 拟合

对每 condition 的三个时点 $(t_i, \hat\sigma_Y(t_i))$ 加权拟合：
$$\sigma_Y(t) = \sigma_{Y,\infty}(1 - e^{-t/T_k})$$

权重 = $1 / \sigma^2_{\text{Laplace}}(t_i)$。报告 $(\hat T_k, \hat\sigma_{Y,\infty})$ + $1\sigma$ 区间。

---

## 9. Pre-registered pass/fail 判据（**不可改！**）

> 这一节在**采任何视频之前**就要打印贴到实验台上。

### C-i — Smooth-material plateau anchor
**适用**：katakuriko 3/5/7%、shiratama 7%
**判据**：$\hat\sigma_{Y,\infty}$ 落在 24 h 平板流变仪 plateau 的 $[0.5\times, 2\times]$ 内（即 $\log_{10}$ 空间内 $\pm 0.3$）

### C-ii — Concentration monotonicity
**适用**：oats 5% vs 9%；katakuriko 3% vs 5% vs 7%
**判据**：$\hat\sigma_{Y,\infty}(\text{higher c}) > \hat\sigma_{Y,\infty}(\text{lower c})$（不允许 1σ overlap）

### C-iii — Cross-batch consistency
**适用**：katakuriko 5%（双 batch）、oats 9%（双 batch）
**判据**：两 batch 的 $(\hat T_k, \hat\sigma_{Y,\infty})$ 在 1σ Laplace CI 内重叠

### C-iv — Kinetic monotonicity
**适用**：全部 7 个 condition
**判据**：$\hat\sigma_Y(40) \ge \hat\sigma_Y(20) \ge \hat\sigma_Y(0)$，允许 1σ overlap

### 失败处理
- 任何判据 fail → **不**追加新实验试图救
- 在 §7.5 Layer C 文字里**显式承认**该 condition fail，并给一个简短的物理解释（例如 "pancake mix 70% 违反 C-iv，可能因 thixotropic recovery 主导"）
- aggregate pass rate = $X / Y$（其中 $Y$ = 适用条件数之和），写进 caption

---

## 10. 风险与常见故障

| 现象 | 可能原因 | 对策 |
|---|---|---|
| 释放瞬间不清晰，silhouette 提取头几帧错位 | 前面板抽起太慢/不平 | 重拍该 session 的所有时点；记入 log "first capture redone" |
| Inverse θ̂_1 与历史值差几个量级 | silhouette 提取漏掉某帧 / 容器规格错 | 检查 video 和 容器尺寸，重跑 inverse；若 still 异常，重拍 |
| Pancake mix 70% 太稠不流动 | 水温偏低 / 粉末过期 | 用新一袋 pmix，水温升至 26 °C |
| Katakuriko 5% 在 t=40 流动比 t=0 还快 | batch 凝胶失败（淀粉受潮）| 该 batch 报废，重做该 condition |
| 双 batch session 第二次粉末用量与第一次差 > 1 g | 称量误差 | 该 batch 报废，重称 |
| ChArUco reprojection error > 2 px | 板被碰移过 | 重做校准；视情况重拍当 session |
| 240 fps 视频出现压缩花纹 | 手机存储不足 | session 开始前清空相册并备份 |
| 室温在某时点偏离 25 ± 0.5 °C | 空调跳停 / 阳光直射 | log 写明该 session 异常；若 > 1 °C 偏差，该 session 报废 |

---

## 11. Layer A / B 合成实验（仅供参考）

这两层不需要 lab。在第 9 节判据 commit 前，可独立先跑：

### Layer A 脚本要求
```bash
python scripts/freshness_layer_A.py \
  --T_k_list 10 20 30 60 120 inf \
  --sigma_Y_inf 100 --n 0.5 --eta 10 \
  --t_points 0 20 40 \
  --n_repeats 5 \
  --out results/fresh/A/
```
产出：`fig:fresh-A1` 用的散点数据 + 计算出的 $T_k^{\min}, T_k^{\max}$。

### Layer B 脚本要求
```bash
python scripts/freshness_layer_B.py \
  --scenarios fast medium slow \
  --T_k_list 15 45 180 \
  --interval_h 8 \
  --n_repeats 10 \
  --out results/fresh/B/
```
产出：`fig:fresh-B1` 用的 $\hat\theta_{\text{fast}}$ vs $\hat\theta_{\text{slow}}$ 双 cloud + 三个 scenario 的 $\Delta$ 数字。

---

## 12. 执行顺序建议

```
Week 1, Day 1-2  : Layer A + Layer B 脚本（合成，纯计算）
                   → 产出 A 的 [T_k^min, T_k^max] 窗口
                   → 验证 §7.5 论证骨架立得住
Week 1, Day 3    : 把 §9 判据打印贴到实验台
Week 2, Day 1    : Lab session × 4（见 §6.4 Day 1）
Week 2, Day 2    : Lab session × 5（见 §6.4 Day 2）
Week 2, Day 3    : Layer D.1 same-batch delayed (1 session)
Week 2, Day 4-5  : 跑所有 joint inverse + Avrami fit
                   → 填 Tab fresh-results
                   → 出图 fig:fresh-traj, fig:fresh-D1
Week 3, Day 1    : 24 h 流变仪测平台（5 个 smooth condition）
Week 3, Day 2    : 整合所有数据，写 Tab fresh-results 解释段
```

如时间压缩到 1 周：A + B 头 1 天跑，§9 判据 day 2 锁定，lab 集中在 day 3-5（不分 day 1/2 慢慢做），day 6-7 分析。

---

## 13. 完成标准

实验"算做完"的判据：

- [ ] Layer A 输出 $T_k^{\min}, T_k^{\max}$ 数值
- [ ] Layer B 输出三个 scenario 的 $\Delta$ 数值
- [ ] Layer C 全部 9 个 session 完成（54 段视频）
- [ ] Layer D.1 完成（2 段视频）
- [ ] Layer D.2 完成（纯分析，无新视频）
- [ ] 24 h 流变仪 plateau 数据完成（4 个 smooth condition）
- [ ] Tab fresh-results 每行填完（含 pass/fail）
- [ ] Aggregate pass rate 数字写好
- [ ] 失败 condition 的物理解释写好

完成后通知论文 §7.5 fill-in 阶段，把数据填入 placeholder 图表。

---

## 附录 A：搅拌动作示意（可省略）

（如需图示再加）

## 附录 B：相机参数推荐

- 分辨率：1080p（更高也行，但 silhouette 提取以 1080p 设计）
- 帧率：240 fps（slo-mo 模式）
- 自动曝光：**关**（手动 ISO 200, shutter 1/240）
- 白平衡：**手动**（5000 K，整个 session 不变）
- 自动对焦：**关**（手动对焦在容器中部，整个 session 不变）

理由：自动模式在 session 内可能漂移，会让 silhouette 提取阈值跨时点不一致。
