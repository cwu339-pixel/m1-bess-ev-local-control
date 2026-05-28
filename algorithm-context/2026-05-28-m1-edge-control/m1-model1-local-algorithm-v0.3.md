# M1 Model 1 本地算法说明 v0.3

日期：2026-05-28

用途：根据 18:00 讨论，把 v0.2 进一步收窄成 Model 1 的简化版本。重点是先把三层架构、Model 1 范围、云端参数、本地输出讲清楚。

---

## 0. 这版为什么改

v0.2 已经把电价、SOC band、Ning draft 02 的 `physical cap × utilization` 结构放进来了。

但 18:00 讨论后，需要进一步简化：

```text
不要把 L1 / L2 / L3 画成一条串行流程。
它们应该是三个 layer，并行运行，通过优先级和 limit 交互。

Model 1 先不要做完整放电经济调度。
放电先理解为 EV 事件驱动的聚合动作，具体比例后续讨论。

Model 1 先把充电侧经济调度做清楚。
```

所以 v0.3 的定位是：

```text
Model 1 = BESS + EV + Import Only 的第一版本地经济调度模型
```

---

## 1. 三层架构

M1 本地控制先按三层理解：

| Layer | 名称 | 做什么 | 责任 |
|---|---|---|---|
| L3 | 云端 advisory / model layer | 下发 model、SOC band、价格 rank、调参参数 | 云端 |
| L2 | 本地协调 / 经济调度 layer | 在 L1 给出的边界内算本地目标 | 我们当前算法 |
| L1 | hard gate / protection layer | MIC / BMS / PCS / SOC / 数据新鲜度硬保护 | 已有 EMS / BMS / PCS / MIC |

关键关系：

```text
L1 不等 L2 跑完才工作。
L3 不等 L2 跑完才工作。
三个 layer 是并行存在的。
```

L1 优先级最高。L1 给 L2 的不是策略，而是：

```text
能不能动？
最多能充多少？
最多能放多少？
数据是否可信？
```

L2 只能在 L1 给出的边界里跑。

---

## 2. Model 1 范围

当前只讨论 Model 1：

```text
model = M1_BESS_EV_IMPORT_ONLY
```

范围：

```text
BESS + EV
Import Only
无 PV
无 export
无 V2G
```

Model 1 先解决：

```text
什么时候给 BESS 充电？
充多少？
EV pool 总功率上限是多少？
```

Model 1 暂不解决：

```text
完整放电经济调度
卖电 / export
V2G
每把枪怎么分配功率
PCS / BMS 底层协议
```

放电在 Model 1 里先按这个口径处理：

```text
EV 到来是事件驱动。
BESS 放电支援 EV / AC 聚合负荷，是功率聚合动作。
第一版不把放电做成和充电完全对称的 price-rank 经济调度公式。
后续再按 SOC band 讨论放电比例。
```

---

## 3. L3 云端下发什么

L3 下发的是本地模型参数，不是底层硬件 command。

### 3.1 model

第一字段建议是：

```text
model = M1_BESS_EV_IMPORT_ONLY
```

后面如果有新的场景，再变成：

```text
model = M2_...
model = M3_...
```

这样本地知道当前运行的是哪套控制逻辑。

### 3.2 MIC / import 参数

`MIC` 可以来自本地配置，也可以由云端下发更新。

原因是：

```text
电网容量或站点 import 合约可能变化。
如果 MIC 变化，需要能通过云端或配置更新到本地。
```

建议字段：

```text
MIC_kw
margin
```

其中：

```text
margin 是比例，例如 0.05 或 0.10
```

具体默认值需要 Ning / IT 确认。

### 3.3 SOC band

当前 SOC 是 BMS 给出的确定值。

SOC band 是云端 probability forecast / model 算出来的边界：

```text
soc_p10
soc_p25
soc_p50
soc_p75
soc_p90
```

本地用当前确定的 `SOC` 去定位它落在哪个 band。

### 3.4 价格 rank

本地不吃原始电价。

云端把价格处理成 rank：

```text
buy_rank_t
```

含义：

```text
0 = 当前买电便宜
1 = 当前买电贵
```

Model 1 先把价格用于 BESS 充电决策。

放电侧的 `spread_rank_t` 暂时不放进 Model 1 主公式，先作为后续讨论项。

### 3.5 充电调参参数

Model 1 先只保留充电侧参数：

```text
f1_chg[band]
g_chg[band]
beta_max_chg_kw
slew_kw
```

含义：

| 参数 | 含义 |
|---|---|
| `f1_chg[band]` | 基础充电利用率，不看价格 |
| `g_chg[band]` | 价格奖励系数，便宜时多用一点 headroom |
| `beta_max_chg_kw` | BESS / PCS 充电硬上限 |
| `slew_kw` | BESS 目标功率变化速度限制 |

具体数值先不在本文定死。

---

## 4. L1 给 L2 什么

L1 是 hard gate，不由 Model 1 重写。

L1 给 L2 的最小结果：

```text
hard_gate_ok
data_fresh
site_safe
bess_charge_limit_kw
bess_discharge_limit_kw
safe_ev_limit_kw
```

如果 L1 不通过：

```text
mode = SAFE_PROTECT
p_bess_target_kw = 0
p_ev_limit_kw = safe_ev_limit_kw
```

如果 L1 通过，L2 才运行 Model 1。

---

## 5. L2 本地实时输入

L2 每个 tick 读取：

```text
SOC
site_load
EV_request
prev_p_bess_kw
```

说明：

```text
SOC 是 BMS 当前确定值
site_load 是站点 AC 聚合负荷
EV_request 是 EV pool 当前请求功率
prev_p_bess_kw 是上一周期 BESS 实际目标或反馈
```

`site_load` 是否需要 EMA 平滑，是实现细节。它可以放在 L2 implementation 中，但不是 Model 1 第一版的核心经济参数。

---

## 6. Model 1 核心公式

### 6.1 电网 headroom

```text
H_t = max(0, (MIC_t - site_load_t) × (1 - margin_t))
```

如果 IT 已经提供 kW 余量，也可以改成：

```text
H_t = max(0, MIC_t - MIC_margin_kw_t - site_load_t)
```

两种方式二选一。

### 6.2 EV 优先后的 BESS 可充上限

```text
alpha_chg_t = max(0, min(
    H_t - EV_request_t,
    bess_charge_limit_kw,
    beta_max_chg_kw
))
```

含义：

```text
先满足 EV 当前请求。
剩余 headroom 才给 BESS 主动充电。
```

所以 Model 1 当前默认：

```text
EV 优先
BESS 不抢 EV 的电
```

如果 SOC 极低时要让 BESS 抢一部分 EV headroom，需要 Ning 单独确认。

### 6.3 SOC band

```text
if   SOC < soc_p10: band = 0
elif SOC < soc_p25: band = 1
elif SOC < soc_p50: band = 2
elif SOC < soc_p75: band = 3
elif SOC < soc_p90: band = 4
else:               band = 5
```

### 6.4 充电使用比例

```text
if band == 0:
    u_chg = 1.0
elif band == 5:
    u_chg = 0
else:
    u_chg = f1_chg[band] + g_chg[band] × (1 - buy_rank_t)
    u_chg = min(u_chg, 1.0)
```

关键点：

```text
价格只影响使用比例。
价格不会额外增加 kW。
```

### 6.5 BESS 充电目标

```text
P_chg_t = alpha_chg_t × u_chg_t

p_bess_target_kw_t = clip(
    P_chg_t,
    prev_p_bess_kw - slew_kw,
    prev_p_bess_kw + slew_kw
)

p_bess_target_kw_t = min(
    p_bess_target_kw_t,
    alpha_chg_t
)
```

在 Model 1 的充电经济调度里：

```text
p_bess_target_kw_t >= 0
```

如果后续加入 EV 事件驱动放电，则负数放电由放电/聚合模块单独定义。

---

## 7. 放电暂时怎么处理

Model 1 先不做完整放电经济调度。

当前口径：

```text
EV 来了以后，BESS 放电支援是事件驱动的聚合动作。
不是 Model 1 的 price-rank 经济调度动作。
```

后续需要单独讨论：

```text
不同 SOC band 下，BESS 给 EV / AC 聚合负荷支援多少比例？
PCS 要求多少功率？
是否由功率聚合环直接给 L2 一个 discharge request？
```

可以预留一个后续形式：

```text
P_dis_event_t = min(
    EV_gap_t,
    bess_discharge_limit_kw
) × r_dis_by_soc_band[band]
```

但 `r_dis_by_soc_band` 这一版不定死。

---

## 8. 输出 Y_t

输出仍保持四个字段：

```js
Y_t = {
  mode,
  p_ev_limit_kw,
  p_bess_target_kw,
  reason_code
}
```

字段含义：

| 字段 | 含义 |
|---|---|
| `mode` | 当前策略状态 |
| `p_ev_limit_kw` | EV pool 总功率上限 |
| `p_bess_target_kw` | BESS 目标功率，Model 1 充电侧为正数或 0 |
| `reason_code` | 当前主原因 |

`p_ev_limit_kw` 仍然是：

```text
P_gun_pool_max
```

每把枪怎么分，不在 Model 1 里做。

---

## 9. 这版相对 v0.2 删掉了什么

为了和 Model 1 对齐，v0.3 暂时拿掉：

```text
f1_dis / g_dis
beta_max_dis_kw 作为经济调度参数
spread_rank_t 作为主公式输入
完整对称放电公式
EV_gap hysteresis 作为主模型参数
site_load EMA 作为主模型参数
allow_buy_grid
```

这些不是永远不要，而是：

```text
先不放进 Model 1 主公式。
等 Ning / IT 确认后，再逐步加。
```

---

## 10. 待确认问题

1. **Model 1 是否只做充电侧经济调度？**

当前 v0.3 按“是”处理。

2. **EV 事件驱动放电比例怎么定？**

是否按 SOC band 给一个比例表：

```text
r_dis_by_soc_band[band]
```

3. **MIC 是云端下发还是本地配置？**

建议都支持，但要明确主口径。

4. **margin 默认值是多少？**

讨论中提到可能不是 0.10，需要 Ning 确认。

5. **f1_chg / g_chg 的默认值谁定？**

当前不使用临时数字作为最终值。

6. **site_load 平滑和 EV_gap 迟滞是否放到 implementation detail？**

当前 v0.3 把它们从主模型里拿掉，只保留为后续实现优化。

---

## 11. 对外口径

可以这样说：

> 这版 v0.3 是在 v0.2 基础上进一步收窄后的 Model 1。我们把 L1 / L2 / L3 拆成三个 layer，L1 负责 hard gate，L3 下发 model、SOC band、price rank 和参数，L2 只在 L1 给出的边界内运行本地经济调度。Model 1 先只把 BESS 主动充电侧做清楚：先算 EV 优先后的物理 headroom，再根据 SOC band 和 buy_rank 算使用比例，最后得到 BESS 充电目标。放电暂时不做完整经济调度，先作为 EV 事件驱动的聚合动作，后续再按 SOC band 商量比例。

