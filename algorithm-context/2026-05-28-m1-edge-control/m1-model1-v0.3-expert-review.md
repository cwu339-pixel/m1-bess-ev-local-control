# M1 Model 1 v0.3 专家评审摘要

日期：2026-05-28

评审对象：

```text
docs/2026-05-28-m1-model1-local-algorithm-v0.3.md
```

评审目标：

```text
判断 v0.3 是否能发给 Ning / IT 看，以及哪些地方必须改到“能解释、能测试、接口闭合”。
```

---

## 1. 总结论

v0.3 的方向是可 defend 的：

```text
Model 1 范围切得足够小。
L1 / L2 / L3 分层是合理的。
输出 Y 仍保持四个字段，适合和 IT 对接口。
Ning draft 02 的“物理上限 × 使用比例”已经被吸收进来。
```

但原始 v0.3 不建议原样发送，因为有几个点别人一问就会卡住：

```text
site_load 口径不清，会导致 EV 被重复扣减。
L1 是否只做前置 gate 没说清。
放电 placeholder 容易被 IT 误解成现在就要实现。
部分变量名太像 AI / 数学稿，不适合会议口头解释。
```

---

## 2. 已经吸收进新版 v0.3 的修改

| 专家意见 | 已做修改 |
|---|---|
| `site_load` 可能包含 EV，导致公式双重扣 EV | 改成 `site_base_load_kw`，明确不含本轮 EV request / BESS target；同时建议 IT 直接给 `site_import_headroom_kw` |
| L1 不应只是前置 gate | 增加“L1 在 L2 输出后还要最终 clamp / reject” |
| 公式里的 `grid_headroom_kw` 太抽象 | 主公式改用 `site_import_headroom_kw` |
| `base_charge_ratio_by_band` / `price_charge_ratio_by_band` 不够清楚 | 改成 `base_charge_ratio_by_soc_band` / `cheap_price_bonus_ratio_by_soc_band` |
| L3 参数缺少有效期 | 增加 `parameter_version` 和 `valid_from / valid_to` |
| 放电公式容易被误解为本版交付 | 明确写成 `future parameter`、`not implemented unless separately enabled` |
| mode 优先级不明确 | 增加 `SAFE_PROTECT > EV_LIMIT > BESS_SUPPORT > BESS_CHARGE > NORMAL` |
| 用户需要能解释 | 新增 `2026-05-28-m1-model1-explainer-v0.3.md` 解说版 |

---

## 3. 当前仍需确认的问题

这些不是 v0.3 的阻塞点，但需要在和 Ning / IT 沟通时确认：

| 问题 | 为什么要确认 |
|---|---|
| IT 能否直接提供 `site_import_headroom_kw` | 能直接提供就避免 `site_base_load` 是否含 EV/BESS 的口径争议 |
| `mic_margin_ratio` 默认值 | 这是 MIC 安全余量，不应由算法文档随便定死 |
| SOC band 是否由云端模型直接输出 | 目标接口按 `soc_p10/p25/p50/p75/p90` 设计 |
| `base_charge_ratio_by_soc_band` / `cheap_price_bonus_ratio_by_soc_band` 初始值 | 这是 Ning / 云端调参项，不应由 L2 自己拍脑袋 |
| EV 事件驱动放电比例 | 本版只预留接口，后续可定义 `discharge_ratio_by_band` |
| IT 最终消费 signed target 还是拆分充/放电字段 | 当前主输出用 signed `p_bess_target_kw` |

---

## 4. 给 Ning / IT 的推荐口径

可以这样说：

> Model 1 v0.3 先定义 Import Only、BESS + EV 场景下的本地充电侧调度：L3 下发模型参数和价格/SOC 指导，L1 提供硬保护边界并最终限幅，L2 在这些边界内计算 EV pool limit 和 BESS 充电目标；放电支援 EV 先作为事件驱动能力预留，不进入本版经济调度公式。

---

## 5. 评审后的判断

这版可以作为下一轮沟通底稿。

但要注意：

```text
它不是完整 EMS。
它不是底层 PCS / charger command。
它是 L2 本地协调算法的接口和公式说明。
```

如果对方问“这个能不能测试”，答案是：

```text
可以。
只要 IT 提供 L3 参数、L1 安全边界、本地实时状态，就能用样例输入跑出 Y_t。
```

