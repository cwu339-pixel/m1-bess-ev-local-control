# M1 MVP 本地算法公式 v0.1

日期：2026-05-28  
目标：先给出一个可以测试的 M1 本地算法。  
场景：`BESS + EV`，Import Only，无 PV、无 export、无 V2G。

---

## 1. 一句话版本

M1 本地算法每 `delta_t` 秒运行一次：

```text
先过安全 gate
再判断 SOC 在哪个 band
再算 EV + AC 聚合负荷有没有缺口
BESS 能补就补
补不了就限制 EV
没有缺口且 SOC 低、云端允许时，才给 BESS 充电
```

核心输出：

```text
p_ev_limit_kw        EV 站级总功率上限
p_bess_target_kw     BESS 目标功率
mode                 当前动作状态
reason_code          为什么这么做
```

---

## 2. 算法需要的数据

### 2.1 云端给本地的数据

| 字段 | 建议粒度 | 含义 | 算法用途 |
|---|---:|---|---|
| `soc_p10` | 30min | SOC 很低参考线 | 低于它原则上不放电 |
| `soc_p25` | 30min | SOC 偏低参考线 | 低于它可考虑充电 |
| `soc_p50` | 30min | SOC 中位参考线 | 后续优化目标中心 |
| `soc_p75` | 30min | SOC 偏高参考线 | 高于它不主动充电 |
| `soc_p90` | 30min | SOC 很高参考线 | 高于它禁止充电 |
| `allow_buy_grid` | 30min | 是否允许买电充 BESS | 控制 BESS 是否可从电网充电 |
| `p_mic_import_kw` | 站点配置 / 30min | import 上限 | 计算 MIC headroom |
| `p_mic_margin_kw` | 站点配置 / 30min | MIC 安全余量 | 防止贴边运行 |
| `cloud_valid_to` | 每包 | 云端数据有效期 | 过期后进入 fallback |

### 2.2 IT / 本地实时数据

| 字段 | 建议粒度 | 含义 | 算法用途 |
|---|---:|---|---|
| `actual_soc` | 1s 或 3s | 当前 BESS SOC | 判断 SOC band / 安全边界 |
| `ev_request_power_kw` | 1s 或 3s | EV 当前总请求功率 | 计算 EV 需求 |
| `ev_actual_power_kw` | 1s 或 3s | EV 当前实际总功率 | 回传和校验 |
| `site_ac_load_kw` | 1s 或 3s | 站内 AC 聚合负荷 | 扣掉 AC 后算 EV headroom |
| `grid_import_power_kw` | 1s 或 3s | 当前电网取电功率 | MIC gate / 校验 |
| `p_gun_pool_max_kw` | 1s 或配置 | 枪池总功率上限 | EV 输出上限不能超过它 |

### 2.3 安全保护数据

| 字段 | 建议粒度 | 含义 | 算法用途 |
|---|---:|---|---|
| `soc_min` | 配置 / 快环 | SOC 安全下限 | 低于它禁止放电 |
| `soc_max` | 配置 / 快环 | SOC 安全上限 | 高于它禁止充电 |
| `bms_status` | 500ms 或设备最快 | BMS 状态 | 故障则进入 protect |
| `pcs_status` | 500ms 或设备最快 | PCS 状态 | 不可用则进入 protect |
| `pcs_charge_limit_kw` | 500ms 或设备最快 | PCS 当前最大充电功率 | 限制 BESS 充电 |
| `pcs_discharge_limit_kw` | 500ms 或设备最快 | PCS 当前最大放电功率 | 限制 BESS 放电 |
| `bms_charge_limit_kw` | 500ms 或设备最快 | BMS 当前允许充电功率 | 限制 BESS 充电 |
| `bms_discharge_limit_kw` | 500ms 或设备最快 | BMS 当前允许放电功率 | 限制 BESS 放电 |
| `data_age_seconds` | 1s 或 3s | 数据延迟 | 判断数据是否新鲜 |
| `data_stale_threshold_seconds` | 配置 | 数据过期阈值 | 超过则 fallback / protect |

---

## 3. 符号定义

每 `delta_t` 秒执行一次。

```text
t                      当前时刻
delta_t                控制周期，单位小时；1 秒 = 1/3600，3 秒 = 3/3600

P_mic                  p_mic_import_kw
P_margin               p_mic_margin_kw
P_ac(t)                site_ac_load_kw
P_ev_req(t)            ev_request_power_kw
P_pool(t)              p_gun_pool_max_kw
SOC(t)                 actual_soc
```

BESS 目标功率建议使用 signed 形式，方便和硬件对齐：

```text
p_bess_target_kw > 0   BESS 充电
p_bess_target_kw = 0   BESS 不动
p_bess_target_kw < 0   BESS 放电
```

如果 IT / PCS 不想用 signed，也可以改成：

```text
mode + abs(p_bess_target_kw)
```

---

## 4. Step 1：安全 gate

先判断关键数据是否可用。

```text
data_fresh(t) =
  data_age_seconds <= data_stale_threshold_seconds
```

基础安全 gate：

```text
base_gate(t) =
  data_fresh(t)
  AND bms_status is OK
  AND pcs_status is OK
```

如果：

```text
base_gate(t) = false
```

则直接输出：

```text
mode = SAFE_PROTECT
p_bess_target_kw = 0
p_ev_limit_kw = safe_ev_limit_kw
reason_code = BESS_FAULT / DATA_STALE
```

这里 `safe_ev_limit_kw` 需要和 IT / 硬件确认。第一版可以先保守写成：

```text
safe_ev_limit_kw = min(P_pool(t), max(0, P_mic - P_margin - P_ac(t)))
```

SOC 硬边界作为方向性限制处理：

```text
SOC(t) <= soc_min  -> 禁止 BESS 放电
SOC(t) >= soc_max  -> 禁止 BESS 充电
```

也就是说，SOC 到下限时不放电，但如果 BMS / PCS 允许，仍然可以充电；SOC 到上限时不充电，但必要时仍然可以放电。

---

## 5. Step 2：SOC band 判断

根据云端下发的 SOC band，计算当前 SOC 状态。

```text
if SOC(t) <= soc_p10:
  soc_band_state = very_low
elif SOC(t) <= soc_p25:
  soc_band_state = low
elif SOC(t) < soc_p75:
  soc_band_state = normal
elif SOC(t) < soc_p90:
  soc_band_state = high
else:
  soc_band_state = very_high
```

动作倾向：

| `soc_band_state` | 动作倾向 |
|---|---|
| `very_low` | 不放电，优先保护 |
| `low` | 少放电；无 EV 缺口且云端允许时可充电 |
| `normal` | 按现场 EV + AC 负荷处理 |
| `high` | 可以放电支援负荷 |
| `very_high` | 不充电；必要时可放电支援负荷 |

---

## 6. Step 3：算 EV + AC 聚合负荷缺口

先算扣掉 AC 负荷后，电网还能给 EV 的功率：

```text
H(t) = max(0, P_mic - P_margin - P_ac(t))
```

其中：

```text
H(t) = grid headroom for EV
```

再算 EV 缺口：

```text
G(t) = max(0, P_ev_req(t) - H(t))
```

其中：

```text
G(t) = EV 请求里，电网余量满足不了的部分
```

---

## 7. Step 4：算 BESS 当前可用能力

### 7.1 可放电能力

如果 SOC 太低，或者安全 gate 不通过，则不允许放电：

```text
discharge_allowed(t) =
  base_gate(t)
  AND SOC(t) > max(soc_min, soc_p10)
```

可放电功率：

```text
P_dis_cap(t) =
  if discharge_allowed(t):
    min(pcs_discharge_limit_kw, bms_discharge_limit_kw)
  else:
    0
```

BESS 用来补 EV 缺口的放电功率：

```text
P_bess_dis(t) = min(G(t), P_dis_cap(t))
```

### 7.2 可充电能力

M1 没有 PV，所以 BESS 从电网充电必须满足：

```text
EV 没有缺口
SOC 偏低
云端允许买电充电
安全 gate 通过
```

写成公式：

```text
charge_allowed(t) =
  base_gate(t)
  AND G(t) = 0
  AND SOC(t) <= soc_p25
  AND SOC(t) < min(soc_max, soc_p90)
  AND allow_buy_grid = true
```

先算满足 AC 和 EV 后，电网还能给 BESS 充电的余量：

```text
R_chg(t) = max(0, P_mic - P_margin - P_ac(t) - P_ev_req(t))
```

可充电功率：

```text
P_chg_cap(t) =
  if charge_allowed(t):
    min(pcs_charge_limit_kw, bms_charge_limit_kw, R_chg(t))
  else:
    0
```

BESS 充电功率：

```text
P_bess_chg(t) = P_chg_cap(t)
```

---

## 8. Step 5：计算输出 y

### 8.1 EV 站级总功率上限

```text
p_ev_limit_kw(t) = min(
  P_pool(t),
  P_ev_req(t),
  H(t) + P_bess_dis(t)
)
```

解释：

```text
EV 最多能拿到的功率
= 电网当前能给 EV 的功率 + BESS 当前能补的功率
但不能超过枪池上限
也不能超过 EV 自己请求的功率
```

### 8.2 BESS 目标功率

用 signed 形式：

```text
p_bess_target_kw(t) = P_bess_chg(t) - P_bess_dis(t)
```

所以：

```text
p_bess_target_kw > 0  充电
p_bess_target_kw = 0  hold
p_bess_target_kw < 0  放电
```

由于充电条件要求 `G(t)=0`，放电条件要求 `G(t)>0`，所以第一版天然不会同时充电和放电。

---

## 9. mode 和 reason_code

mode 可以按下面规则生成。

```text
if base_gate(t) = false:
  mode = SAFE_PROTECT

elif cloud_valid_to expired:
  mode = SAFE_FALLBACK

elif G(t) > 0 and P_bess_dis(t) >= G(t):
  mode = BESS_DISCHARGE

elif G(t) > 0 and 0 < P_bess_dis(t) < G(t):
  mode = BESS_DISCHARGE_AND_EV_LIMIT

elif G(t) > 0 and P_bess_dis(t) = 0:
  mode = EV_LIMIT_ONLY

elif P_bess_chg(t) > 0:
  mode = BESS_CHARGE

else:
  mode = BESS_HOLD
```

reason_code 第一版可以这样给：

| 场景 | `reason_code` |
|---|---|
| BMS / PCS 故障 | `BESS_FAULT` |
| 数据过期 | `DATA_STALE` |
| 云端 guidance 过期 | `CLOUD_EXPIRED` |
| EV 请求超过本地余量 | `EV_DEMAND_HIGH` |
| MIC 余量不足 | `MIC_LIMIT` |
| PCS 限制导致 BESS 补不满 | `PCS_LIMIT` |
| BMS 限制导致 BESS 补不满 | `BMS_LIMIT` |
| SOC 太低不能放电 | `SOC_LOW` |
| SOC 正常且无缺口 | `SOC_IN_BAND` |

---

## 10. 输出 schema

MVP 不要把输出做复杂。

真正给 IT / 硬件执行的控制输出只有两个：

```js
local_decision_output = {
  timestamp,
  p_ev_limit_kw,
  p_bess_target_kw
}
```

字段含义：

| 字段 | 含义 |
|---|---|
| `timestamp` | 本轮决策时间 |
| `p_ev_limit_kw` | EV 站级总功率上限 |
| `p_bess_target_kw` | BESS 目标功率，建议 signed |

如果硬件侧还要求“这个 setpoint 有效多久”，再加一个字段：

```js
command_valid_for_seconds
```

第一版可以设成：

```text
command_valid_for_seconds = control_period_seconds
```

也就是算法 1 秒算一次，就有效 1 秒；算法 3 秒算一次，就有效 3 秒。

用于调试和回传云端的解释字段可以保留，但它们不是硬件控制命令：

```js
debug_info = {
  mode,
  reason_code,
  soc_band_state,
  gate_status,
  p_grid_available_kw,
  ev_gap_kw,
  p_bess_support_kw
}
```

所以最终口径是：

```text
控制输出：
  p_ev_limit_kw
  p_bess_target_kw
  command_valid_for_seconds 可选

解释输出：
  mode
  reason_code
  soc_band_state
  gate_status
```

---

## 11. 物理约束

M1 必须满足：

```text
0 <= p_ev_limit_kw(t) <= P_pool(t)
0 <= p_ev_limit_kw(t) <= P_ev_req(t)
0 <= P_bess_dis(t) <= P_dis_cap(t)
0 <= P_bess_chg(t) <= P_chg_cap(t)
P_bess_chg(t) * P_bess_dis(t) = 0
P_ac(t) + p_ev_limit_kw(t) + P_bess_chg(t) - P_bess_dis(t) <= P_mic - P_margin
export_power_kw(t) = 0
soc_min <= SOC(t) <= soc_max
```

SOC 物理更新：

```text
SOC(t+1) = SOC(t)
         + eta_chg * P_bess_chg(t) * delta_t / E_bess
         - P_bess_dis(t) * delta_t / (eta_dis * E_bess)
```

其中：

```text
E_bess      BESS 容量，单位 kWh
eta_chg     充电效率
eta_dis     放电效率
delta_t     控制周期，单位小时
```

---

## 12. 后续如果要优化器

第一版不用优化器，直接按上面的规则公式算。

但如果后续要升级成优化器，可以把决策变量写成：

```text
decision variables:
  p_ev_limit_kw(t)
  P_bess_chg(t)
  P_bess_dis(t)
  SOC(t)
```

约束就是第 11 节的物理约束。

一个简单目标函数可以是：

```text
minimize Σ_t [
  lambda_ev  * (P_ev_req(t) - p_ev_limit_kw(t))^2
  + lambda_soc * (SOC(t) - soc_p50(t))^2
  + lambda_deg * (P_bess_chg(t) + P_bess_dis(t))
  + lambda_mic * max(0, P_ac(t) + p_ev_limit_kw(t) + P_bess_chg(t) - P_bess_dis(t) - (P_mic - P_margin))^2
]
```

人话解释：

```text
尽量满足 EV
尽量让 SOC 靠近云端 soc_p50
尽量少折腾电池
尽量不要贴着 MIC 上限跑
```

所以 MVP 的路线是：

```text
现在：规则公式直接算
以后：同一套变量和约束交给优化器
```

---

## 13. 明天需要 IT 确认的问题

1. `site_ac_load_kw` 是否能提供？是否不包含 EV 充电功率？
2. `ev_request_power_kw` 是站级总请求功率，还是每枪请求后需要本地汇总？
3. `p_gun_pool_max_kw` 是谁给的？EMS、OCPP SiteSupervisor，还是 charger controller？
4. `p_ev_limit_kw` 这个站级总功率上限，IT 能否直接接收并下发？
5. `p_bess_target_kw` 硬件侧希望用 signed，还是 `mode + abs(kW)`？
6. 控制输出是否需要 `command_valid_for_seconds`？如果需要，是否等于算法周期？
7. BMS / PCS 是否能提供实时 charge/discharge limit？
8. 数据刷新是 1 秒、3 秒，还是其他？对应 `data_stale_threshold_seconds` 应该设多少？
9. `SAFE_PROTECT` 时，EV 是否继续按 MIC headroom 限制，还是直接降到固定安全值？
