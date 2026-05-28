# M1 MVP 数据需求表：给金总确认 v0.1

日期：2026-05-28  
用途：确认 M1 本地 MVP 算法需要哪些输入数据、从哪里拿、粒度是多少、输出能否被 IT / 硬件侧接收。

---

## 1. 当前已确定的算法闭环

M1 范围：

```text
BESS + EV
Import Only
无 PV
无 export
无 V2G
```

算法闭环：

```text
X_t 输入数据
  ↓
f(X_t) 本地最小规则算法
  ↓
Y_t 输出给 IT / EMS / 硬件
```

核心公式：

```text
P_grid_available = max(0, MIC - MIC_margin - site_load)

EV_gap = max(0, EV_request - P_grid_available)

BESS_available = min(
  PCS_discharge_limit,
  BMS_discharge_limit,
  SOC_discharge_limit
)

如果 BESS_gate 不通过：
  BESS_available = 0

如果 SOC < soc_p25：
  BESS_available = 0

P_bess_support = min(EV_gap, BESS_available)

如果 EV_site_gate 不通过：
  p_ev_limit_kw = 0
否则：
  p_ev_limit_kw = min(EV_request, P_grid_available + P_bess_support)

p_bess_target_kw = -P_bess_support
```

如果包含 BESS 充电场景：

```text
P_bess_charge = min(
  grid_charge_room,
  PCS_charge_limit,
  BMS_charge_limit,
  SOC_charge_room
)

p_bess_target_kw = P_bess_charge - P_bess_support
```

---

## 2. 最终输出 Y_t

M1 MVP 输出先固定为 4 个字段：

```js
Y_t = {
  mode,
  p_ev_limit_kw,
  p_bess_target_kw,
  reason_code
}
```

| 字段 | 含义 | 给谁用 |
|---|---|---|
| `mode` | 当前本地策略状态 | 日志 / 云端回传 / 调试 |
| `p_ev_limit_kw` | EV pool 最大允许功率，可理解为站级 `P_gun_pool_max` | IT / EMS / 充电侧执行 |
| `p_bess_target_kw` | BESS 目标功率，正数充电，负数放电，0 不动 | BESS / PCS 控制侧执行 |
| `reason_code` | 当前输出主原因 | 日志 / 云端回传 / 调试 |

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

## 3. 数据需求表

### 3.1 云端数据：已固定

云端会给 5 类 30min 时间序列数据。

| 字段 | 是否本地 MVP 必需 | 粒度 | 说明 |
|---|---|---:|---|
| `actual_ev_fee` | 否 | 30min | 收益复盘 / 展示 / 云端模型校验 |
| `actual_grid_cost` | 否 | 30min | 成本复盘 / 展示 / 云端模型校验 |
| `forecast_load` | 否 | 30min | 云端预测用；本地 MVP 仍以实时 `site_load` 为准 |
| `forecast_ev_energy` | 否 | 30min | 云端预测用；本地 MVP 仍以实时 `EV_request` 为准 |
| `expected_soc` / `soc_band` | 是 | 30min | 云端 SOC guidance；MVP 按 SOC band 判断 SOC 是否偏低 |

SOC band 的处理：

```text
云端直接给 soc_p10/p25/p50/p75/p90
本地直接使用
```

MVP 主算法最关键的是：

```text
soc_p25
```

需要确认：

| 字段 | 是否必需 | 来源 | 粒度 | 说明 |
|---|---|---|---:|---|
| `soc_p10` | 否，MVP 可不用 | 云端 SOC band | 30min | 后续可作为更强保护参考 |
| `soc_p25` | 是 | 云端 SOC band | 30min | 低于它时，BESS 不放电支援 EV |
| `soc_p50` | 否，MVP 可不用 | 云端 SOC band | 30min | 后续优化器目标中心 |
| `soc_p75` | 否，MVP 可不用 | 云端 SOC band | 30min | 后续用于判断 SOC 偏高 |
| `soc_p90` | 否，MVP 可不用 | 云端 SOC band | 30min | 后续可作为高 SOC 参考 |

---

### 3.2 站点实时数据：需要 IT / EMS 确认

| 字段 | 是否必需 | 期望来源 | 期望粒度 | 单位 | 说明 |
|---|---|---|---:|---|---|
| `MIC` | 是 | 已有 MIC 模块 / 站点配置 / EMS | 配置或实时 | kW | 站点最大允许从电网取电功率 |
| `MIC_margin` | 是，可先设 0 | 已有 MIC 保护算法 / EMS 配置 | 配置或实时 | kW | MIC 安全余量，防止贴边运行 |
| `site_load` | 是 | 电表 / STS / EMS | 1s 或 3s | kW | 站内 AC 聚合负荷，最好不包含 EV |
| `EV_request` | 是 | 充电桩 / OCPP / SiteSupervisor / CGPower | 1s 或 3s | kW | 当前 EV pool 总请求功率 |
| `EV_site_gate` | 是 | EMS / 充电侧 / 站级保护 | 1s 或事件 | bool | EV / site 是否允许继续运行 |
| `data_fresh` | 是 | EMS / 数据采集层 | 1s 或 3s | bool | 关键数据是否新鲜 |

说明：

```text
MIC 和 MIC_margin 不属于云端固定 5 个字段。
它们应该来自本地站点配置、已有 MIC 模块或 EMS。
```

如果 `site_load` 里已经包含 EV，则公式要调整；所以需要确认：

```text
site_load 是否不包含 EV 充电功率？
```

---

### 3.3 BESS / PCS / BMS 数据：需要确认

| 字段 | 是否必需 | 期望来源 | 期望粒度 | 单位 | 说明 |
|---|---|---|---:|---|---|
| `SOC` | 是 | BMS / EMS | 1s 或 3s | % | 当前电池 SOC |
| `SOC_min` | 是 | BMS / EMS 配置 | 配置 | % | SOC 安全下限，低于它禁止放电 |
| `SOC_max` | 是 | BMS / EMS 配置 | 配置 | % | SOC 安全上限，高于它禁止充电 |
| `BESS_gate` | 是 | 已有 L1 保护 / BMS / PCS / EMS | 1s 或事件 | bool | BESS 当前是否允许参与充放电 |
| `PCS_discharge_limit` | 是，除非已有 `BESS_available` | PCS / EMS | 1s 或 3s | kW | PCS 当前最大可放电功率 |
| `BMS_discharge_limit` | 是，除非已有 `BESS_available` | BMS / EMS | 1s 或 3s | kW | BMS 当前允许最大放电功率 |
| `SOC_discharge_limit` | 是，除非已有 `BESS_available` | BMS / EMS / 本地计算 | 1s 或 3s | kW | 因 SOC 限制当前还能放多少 |
| `BESS_available` | 可替代上面三个 limit | 已有 L1 / EMS | 1s 或 3s | kW | 如果已有模块能直接给 BESS 可支援功率，算法可直接使用 |

BESS 充电场景还需要：

| 字段 | 是否必需 | 期望来源 | 期望粒度 | 单位 | 说明 |
|---|---|---|---:|---|---|
| `PCS_charge_limit` | 仅 BESS_CHARGE 需要 | PCS / EMS | 1s 或 3s | kW | PCS 当前最大可充电功率 |
| `BMS_charge_limit` | 仅 BESS_CHARGE 需要 | BMS / EMS | 1s 或 3s | kW | BMS 当前允许最大充电功率 |
| `SOC_charge_room` | 仅 BESS_CHARGE 需要 | BMS / EMS / 本地计算 | 1s 或 3s | kW | 因 SOC 上限，当前还能充多少 |

BMS 说明：

```text
BMS = Battery Management System，电池管理系统。
它通常负责 SOC、告警、充放电允许状态、充放电限额等。
```

---

### 3.4 输出接口：需要 IT / 硬件确认

| 输出字段 | 是否必需 | 接收方 | 期望粒度 | 单位 | 需要确认 |
|---|---|---|---:|---|---|
| `p_ev_limit_kw` | 是 | EMS / OCPP / SiteSupervisor / 充电侧 | 跟随算法周期，1s 或 3s | kW | 能否作为 EV pool 总功率上限下发 |
| `p_bess_target_kw` | 是 | EMS / PCS / BESS 控制侧 | 跟随算法周期，1s 或 3s | kW | 是否接受 signed kW：正数充电，负数放电 |
| `mode` | 是 | 日志 / 云端 / 调试 | 跟随算法周期 | enum | 是否需要下发给硬件，还是只回传 |
| `reason_code` | 是 | 日志 / 云端 / 调试 | 跟随算法周期 | enum | 是否需要下发给硬件，还是只回传 |

建议：

```text
p_ev_limit_kw 和 p_bess_target_kw 是控制输出。
mode 和 reason_code 先作为日志 / 回传字段。
```

---

## 4. 最小闭环必需字段

如果只为了让 MVP 算法跑起来，最小字段是：

```text
cloud:
  soc_p25
  soc_p50 可选
  soc_p75 可选

site:
  MIC
  MIC_margin
  site_load
  EV_request
  EV_site_gate
  data_fresh

BESS:
  SOC
  SOC_min
  BESS_gate
  BESS_available

output:
  p_ev_limit_kw
  p_bess_target_kw
```

如果没有 `BESS_available`，则需要：

```text
PCS_discharge_limit
BMS_discharge_limit
SOC_discharge_limit
```

---

## 5. 需要金总确认的问题

1. MIC 是不是已有模块 / 算法能提供？输出字段名是什么？
2. MIC_margin 是否已有？如果没有，MVP 能否先设为 0？
3. `site_load` 是否能提供？是否不包含 EV 充电功率？
4. `EV_request` 是否能提供站级总请求功率？如果没有，是否能从多枪功率请求汇总？
5. `SOC`、`SOC_min`、`SOC_max` 是否能提供？
6. 是否已有 `BESS_gate`，表示 BESS 当前能否参与充放电？
7. 是否已有 `BESS_available`，直接表示 BESS 当前可支援功率？
8. 如果没有 `BESS_available`，是否能提供 `PCS_discharge_limit / BMS_discharge_limit / SOC_discharge_limit`？
9. 是否需要支持 BESS_CHARGE？如果需要，是否能提供 `PCS_charge_limit / BMS_charge_limit / SOC_charge_room`？
10. `p_ev_limit_kw` 能不能作为 EV pool 的总功率上限下发？
11. `p_bess_target_kw` 能不能按 signed kW 下发？正数充电，负数放电。
12. 上面这些实时数据的刷新粒度是 1s、3s，还是其他？
