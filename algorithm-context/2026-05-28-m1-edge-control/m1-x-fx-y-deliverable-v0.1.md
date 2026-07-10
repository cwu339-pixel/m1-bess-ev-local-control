# M1 MVP 本地算法说明：X -> f(X) -> Y v0.2

日期：2026-05-28  
目标：打通 M1 本地控制逻辑闭环，明确需要哪些输入 `X_t`，本地算法 `f(X_t)` 怎么算，最终输出 `Y_t` 是什么。

---

## 0. 范围

当前只讨论 M1：

```text
M1 = BESS + EV
Import Only
无 PV
无 export
无 V2G
```

本阶段不讨论完整经济优化、不讨论 per-gun 分配、不讨论最终 PCS / OCPP 协议细节。

MVP 只做一件事：

> 本地 L2 最小规则算法根据云端 SOC guidance、本地实时负荷和已有 L1 硬保护结果，计算 EV pool 最大允许功率和 BESS 目标功率。

---

## 1. 和 PPT 的对应关系

PPT 里的框架可以放进来，但只取 M1 MVP 需要的部分。

| PPT 说法 | 在 M1 MVP 里的落法 |
|---|---|
| L3 云端 advisory | D+1 `cloud_downlink_packet`，30min 粒度，给 SOC guidance，不直接控制硬件 |
| SOC band | 云端直接下发 `soc_p10/p25/p50/p75/p90`，本地直接使用 |
| L2 1s 本地协调 | 本文要交付的最小本地规则算法 `f(X_t)` |
| L1 D 快环 | 已有机器 / EMS / BMS / PCS 的硬保护；M1 只读取 gate 和 limit，不重做保护算法 |
| A) BESS 放电 | BESS 支援 EV + AC 聚合负荷 |
| EMS 只下发 `P_gun_pool_max` | M1 只输出 site-level `p_ev_limit_kw`，不做 per-gun 分配 |

一句话：

```text
L3 给 SOC guidance
L2 用最小规则算 Y_t
L1 已有硬保护给 gate / limit
```

---

## 2. 整体闭环

```text
X_t 输入数据
  ↓
f(X_t) 本地规则算法
  ↓
Y_t 本地输出
```

其中：

```text
X_t = 云端固定下发数据 + 本地站点实时状态 + 已有 L1 gate / limit

f(X_t) = 算 grid available、EV gap、BESS support，再生成 Y_t

Y_t = {
  mode,
  p_ev_limit_kw,
  p_bess_target_kw,
  reason_code
}
```

---

## 3. X_t：MVP 最小输入

### 3.1 A. `cloud_downlink_packet`

这是云端已经确定会给我们的数据。

粒度：

```text
D+1
30min 时间序列
每天更新
```

字段：

| 云端字段 | MVP 本地算法是否直接使用 | 说明 |
|---|---|---|
| `actual_ev_fee` | 暂不直接使用 | 用于收益复盘、展示和模型校验 |
| `actual_grid_cost` | 暂不直接使用 | 用于成本复盘、展示和模型校验 |
| `forecast_load` | 暂不直接使用 | 云端预测用；本地 L2 仍以实时 `site_load` 为准 |
| `forecast_ev_energy` | 暂不直接使用 | 云端预测用；本地 L2 仍以实时 `EV_request` 为准 |
| `expected_soc` / `soc_band` | 使用 | 云端直接下发 SOC guidance；MVP 按 SOC band 使用 |

第一版本地真正用到的是 SOC band：

```text
soc_p10
soc_p25
soc_p50
soc_p75
soc_p90
```

其中 MVP 主算法最关键的是：

```text
soc_p25
```

它用于判断：

```text
如果 SOC < soc_p25：
  BESS 不放电支援 EV
```

`soc_p50` 可作为后续优化器的目标中心，`soc_p75/p90` 后续可用于判断 SOC 偏高或禁止继续充电。

注意：

```text
SOC band 是分类边界，不是硬件实时数据
MIC / MIC_margin 不是云端这 5 个固定字段里的数据
```

### 3.2 B. `realtime_site_state`

这是本地 / IT / EMS 需要提供的实时站点数据。

粒度：

```text
建议 1s 或 3s
实际由 IT / 现场系统确认
```

最小字段：

| 字段 | 含义 | 用途 |
|---|---|---|
| `MIC` | 站点最大允许从电网取电功率 | 计算电网可用功率 |
| `MIC_margin` | MIC 安全余量 | 防止贴着 MIC 上限跑；MVP 可先设为 0 |
| `site_load` | 当前站内 AC 聚合负荷，不含 EV | 从 MIC 中扣除 |
| `EV_request` | 当前 EV pool 总请求功率 | 计算 EV gap |

说明：

```text
MIC 来自站点配置 / IT / EMS / 本地参数
MIC_margin 来自本地配置或已有 MIC 保护逻辑
它们不是云端固定 5 个字段
```

### 3.3 C. `existing_L1_gate_and_limits`

这是已有机器 / EMS / BMS / PCS / 本地保护层提供的 gate 和 limit。

粒度：

```text
设备最快稳定粒度
PPT 中 L1 是 200ms
MVP 实际可以先按现场可提供的 500ms / 1s / 3s
但必须有 data_fresh 判断
```

最小字段：

| 字段 | 含义 | 用途 |
|---|---|---|
| `EV_site_gate` | EV / site 是否允许继续输出功率 | 不通过则 `SAFE_PROTECT` |
| `BESS_gate` | BESS 是否允许参与充放电 | 不通过则 BESS 不支援 |
| `data_fresh` | 关键数据是否新鲜 | 不新鲜则保护或降级 |
| `SOC` | 当前 BESS SOC | 判断 SOC band |
| `SOC_min` | SOC 硬下限 | 到下限禁止放电 |
| `SOC_max` | SOC 硬上限 | 到上限禁止充电 |
| `PCS_discharge_limit` | PCS 当前可放电功率 | 计算 BESS 可支援能力 |
| `BMS_discharge_limit` | BMS 当前可放电功率 | 计算 BESS 可支援能力 |
| `SOC_discharge_limit` | SOC 允许的可放电功率 | 计算 BESS 可支援能力 |

如果已有 L1 能直接给：

```text
BESS_available
```

那可以直接用它，不一定要本地 L2 再算 `min(PCS, BMS, SOC)`。

如果 MVP 要包含 BESS 充电，还需要：

| 字段 | 含义 | 用途 |
|---|---|---|
| `PCS_charge_limit` | PCS 当前可充电功率 | 计算 BESS charge |
| `BMS_charge_limit` | BMS 当前可充电功率 | 计算 BESS charge |
| `SOC_charge_room` | SOC 剩余可充空间对应功率 | 计算 BESS charge |

---

## 4. f(X_t)：本地 MVP 规则算法

### Step 1：读取已有 gate

M1 不重新实现 L1 硬保护，只读取 L1 已经判断好的结果。

```text
G_site,t = 1{EV_site_gate_t = 1 AND data_fresh_t = 1}

G_bess,t = 1{BESS_gate_t = 1 AND data_fresh_t = 1}
```

人话：

```text
G_site,t = 0  -> EV / site 不安全，EV 输出直接进入 SAFE_PROTECT
G_bess,t = 0  -> BESS 不安全，BESS 不参与支援，但 EV 未必必须停
```

如果：

```text
G_site,t = 0
```

直接输出：

```text
mode = SAFE_PROTECT
p_ev_limit_kw = 0
p_bess_target_kw = 0
```

### Step 2：算电网可用功率

公式：

```text
P_grid_available,t = max(0, MIC_t - MIC_margin_t - site_load_t)
```

人话：

> MIC 扣掉安全余量和当前站内 AC 聚合负荷后，剩下多少功率可以给 EV pool 使用。

### Step 3：算 EV 缺口

公式：

```text
EV_gap_t = max(0, EV_request_t - P_grid_available,t)
```

人话：

> EV 想要的功率减去电网当前还能给 EV 的功率，就是 EV 缺口。

### Step 4：算 BESS 可支援功率

如果 L1 已经直接提供 `BESS_available_t`，则直接使用。

如果 L1 没有直接提供，则 L2 可按下面方式计算：

```text
BESS_available_raw,t = min(
  PCS_discharge_limit_t,
  BMS_discharge_limit_t,
  SOC_discharge_limit_t
)
```

再加 gate 和 SOC band 限制：

```text
BESS_available_t =
  if G_bess,t = 1
     AND SOC_t > SOC_min
     AND SOC_t >= soc_p25_t:
       BESS_available_raw,t
  else:
       0
```

人话：

> BESS 只有在已有 L1 gate 允许，并且 SOC 不低于 p25 时，才允许放电支援 EV + AC 聚合负荷。

### Step 5：算 BESS 支援 EV 的功率

公式：

```text
P_bess_support_t = min(EV_gap_t, BESS_available_t)
```

人话：

> BESS 最多只补 EV 缺口，不多放电；如果 BESS 能力不够，就补一部分。

### Step 6：输出 EV limit

如果 EV / site gate 不通过：

```text
p_ev_limit_kw,t = 0
```

否则：

```text
p_ev_limit_kw,t = min(
  EV_request_t,
  P_grid_available,t + P_bess_support_t
)
```

人话：

> EV pool 最大允许功率 = 电网当前可给 EV 的功率 + BESS 当前支援功率，但不能超过 EV 自己请求的功率。

这里的 `p_ev_limit_kw` 可以对接为 EV pool 的 `P_gun_pool_max`。

### Step 7：输出 BESS target

放电支援场景：

```text
p_bess_target_kw,t = - P_bess_support_t
```

如果是 BESS 充电场景，再计算：

```text
charge_allowed_t =
  1{
    G_site,t = 1
    AND G_bess,t = 1
    AND EV_gap_t = 0
    AND SOC_t <= soc_p25_t
    AND SOC_t < SOC_max
    AND allow_buy_grid_t = 1
  }
```

电网剩余可用于 BESS 充电的功率：

```text
P_charge_room_t = max(
  0,
  MIC_t - MIC_margin_t - site_load_t - EV_request_t
)
```

BESS 充电功率：

```text
P_bess_charge_t =
  if charge_allowed_t = 1:
    min(
      P_charge_room_t,
      PCS_charge_limit_t,
      BMS_charge_limit_t,
      SOC_charge_room_t
    )
  else:
    0
```

最终：

```text
p_bess_target_kw,t = P_bess_charge_t - P_bess_support_t
```

符号约定：

```text
p_bess_target_kw > 0 表示 BESS 充电
p_bess_target_kw < 0 表示 BESS 放电
p_bess_target_kw = 0 表示 BESS 不动
```

MVP 里，放电支援和充电不会同时发生：

```text
EV_gap_t > 0  -> 只考虑 BESS 支援
EV_gap_t = 0  -> 才考虑 BESS 充电
```

---

## 5. mode 判断

mode 只保留 5 个：

```text
NORMAL
BESS_SUPPORT
EV_LIMIT
BESS_CHARGE
SAFE_PROTECT
```

判断顺序：

```text
if G_site,t = 0:
    mode_t = SAFE_PROTECT

elif p_ev_limit_kw,t < EV_request_t:
    mode_t = EV_LIMIT

elif P_bess_support_t > 0:
    mode_t = BESS_SUPPORT

elif P_bess_charge_t > 0:
    mode_t = BESS_CHARGE

else:
    mode_t = NORMAL
```

说明：

> 如果 BESS 已经支援了，但 EV 仍然拿不到请求功率，主 mode 仍然是 `EV_LIMIT`。BESS 是否参与支援，可以从 `p_bess_target_kw < 0` 看出来。

---

## 6. reason_code 判断

reason_code 只保留这些：

```text
NORMAL
EV_DEMAND_HIGH
MIC_LIMIT
SOC_LOW
PCS_LIMIT
BESS_FAULT
DATA_STALE
```

建议第一版按下面优先级判断：

```text
if data_fresh_t = 0:
    reason_code_t = DATA_STALE

elif G_site,t = 0:
    reason_code_t = BESS_FAULT

elif G_bess,t = 0 AND EV_gap_t > 0:
    reason_code_t = BESS_FAULT

elif EV_gap_t > 0 AND SOC_t < soc_p25_t:
    reason_code_t = SOC_LOW

elif EV_gap_t > 0 AND 0 < BESS_available_t < EV_gap_t:
    reason_code_t = PCS_LIMIT

elif p_ev_limit_kw,t < EV_request_t:
    reason_code_t = MIC_LIMIT

elif P_bess_support_t > 0:
    reason_code_t = EV_DEMAND_HIGH

else:
    reason_code_t = NORMAL
```

说明：

> `reason_code` 是主原因，不要求覆盖所有细节。MVP 先够解释为什么限功率、为什么 BESS 不支援即可。

---

## 7. Y_t：最终输出

M1 MVP output 固定为：

```js
y_t = {
  mode,
  p_ev_limit_kw,
  p_bess_target_kw,
  reason_code
}
```

字段解释：

| 字段 | 含义 |
|---|---|
| `mode` | 当前本地策略状态 |
| `p_ev_limit_kw` | EV pool 最大允许功率，可对接为 `P_gun_pool_max` |
| `p_bess_target_kw` | BESS 目标功率，正数充电，负数放电，0 不动 |
| `reason_code` | 当前输出的主原因 |

`mode` 枚举：

```text
NORMAL
BESS_SUPPORT
EV_LIMIT
BESS_CHARGE
SAFE_PROTECT
```

`reason_code` 枚举：

```text
NORMAL
EV_DEMAND_HIGH
MIC_LIMIT
SOC_LOW
PCS_LIMIT
BESS_FAULT
DATA_STALE
```

---

## 8. 上传口径

第一版上传云端也先用同一个 `Y_t`。

```js
cloud_uplink_report = {
  timestamp,
  mode,
  p_ev_limit_kw,
  p_bess_target_kw,
  reason_code
}
```

如果后续云端需要复盘，再补充输入快照，例如：

```text
MIC_t
site_load_t
EV_request_t
SOC_t
P_grid_available_t
EV_gap_t
P_bess_support_t
```

但这些不是 MVP 控制输出。

---

## 9. 粒度汇总

| 数据 / 动作 | MVP 粒度 | 说明 |
|---|---:|---|
| `cloud_downlink_packet` | D+1，30min 时间序列，每天更新 | 云端固定 5 个字段，主要用 `expected_soc` |
| SOC band | 30min | 云端直接下发 `soc_p10/p25/p50/p75/p90` |
| `realtime_site_state` | 1s 或 3s | IT / EMS 提供现场实时数据 |
| `existing_L1_gate_and_limits` | 设备最快稳定粒度；目标 500ms~1s | 已有硬保护结果，M1 只读取 gate / limit |
| `f(X_t)` 本地规则算法 | 1s 或 3s | L2 本地协调 |
| `Y_t` 输出 | 跟随算法周期 | 输出给 IT / EMS / 硬件控制链路 |
| 故障 / 保护事件上传 | 立即 | BMS/PCS/MIC/data stale 等事件 |
| 普通复盘上传 | 30min 或 daily | 用于展示、模型校验和后续优化 |

---

## 10. 最小压缩版公式

如果只看核心公式，可以压缩成：

```text
从云端 SOC band 读取 soc_p25_t

G_site,t = 1{EV_site_gate_t = 1 AND data_fresh_t = 1}

G_bess,t = 1{BESS_gate_t = 1 AND data_fresh_t = 1}

P_grid_available,t = max(0, MIC_t - MIC_margin_t - site_load_t)

EV_gap_t = max(0, EV_request_t - P_grid_available,t)

BESS_available_t =
  if G_bess,t = 1 AND SOC_t > SOC_min AND SOC_t >= soc_p25_t:
    min(PCS_discharge_limit_t, BMS_discharge_limit_t, SOC_discharge_limit_t)
  else:
    0

P_bess_support_t = min(EV_gap_t, BESS_available_t)

p_ev_limit_kw,t =
  if G_site,t = 0:
    0
  else:
    min(EV_request_t, P_grid_available,t + P_bess_support_t)

p_bess_target_kw,t = P_bess_charge_t - P_bess_support_t
```

输出：

```js
y_t = {
  mode_t,
  p_ev_limit_kw_t,
  p_bess_target_kw_t,
  reason_code_t
}
```

---

## 11. 需要 IT / 硬件确认

1. `site_load_t` 是否能提供，并且是否不包含 EV 充电功率？
2. `EV_request_t` 是站级 EV pool 总请求功率，还是需要本地从多枪汇总？
3. `MIC_t` 和 `MIC_margin_t` 由谁提供？是固定配置、EMS 参数，还是已有 MIC 保护模块输出？
4. `p_ev_limit_kw` 是否可以直接作为 EV pool 的 `P_gun_pool_max` 下发？
5. `p_bess_target_kw` 硬件侧是否接受 signed kW？
6. `BESS_gate` 和 `EV_site_gate` 是否已有？由谁提供？
7. `PCS_discharge_limit_t`、`BMS_discharge_limit_t`、`SOC_discharge_limit_t` 能否直接提供？如果已有 `BESS_available_t`，能否直接提供？
8. 数据粒度实际是 1s、3s，还是其他？
9. 云端下发的 SOC band 是否包含 `soc_p10/p25/p50/p75/p90`？MVP 至少需要 `soc_p25`。
