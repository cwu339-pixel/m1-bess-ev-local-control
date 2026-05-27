# M1 本地控制接口 v0.1 交付稿

日期：2026-05-26

用途：用于产品、云端、IT 和硬件侧对齐。目标是确认 M1 的问题定义、约束、参数列表、状态/动作空间和测试方式。不是最终算法，也不是精确 kW dispatch。

## 0. 这版只讲三部分

这版不要再发散，只讲清楚三件事：

```text
1. 输入状态：云端参数 + 本地实时状态
2. 硬约束：哪些动作不能做
3. 输出动作状态：mode + target + reason_code
```

核心关系：

```text
云端参数 + 本地实时状态
        ↓
形成当前状态空间 x_t
        ↓
硬约束 K 过滤掉不可行动作
        ↓
在可行动作里选择一个输出
        ↓
decision = {mode, target, reason_code}
```

注意：

> 硬约束不是输出。硬约束是过滤器。它决定哪些动作不能做。最后输出的是本地动作状态，例如充电、放电、待机、限功率、保护、fallback。

可以写成：

```text
x_t = 当前现场状态
K = 硬约束集合
U = 所有可能动作

U_feasible(t) = filter(U, x_t, K)

decision_t = select(U_feasible(t))
```

最后：

```text
decision_t = {
  mode_t,
  target_t,
  reason_code_t
}
```

## 1. 问题定义

当前只考虑 M1：

```text
M1 = BESS + EV
Import Only
无 PV
无 export
无 V2G
```

要解决的问题：

> 云端给半小时粒度参数包，本地结合实时状态和硬约束，输出一个本地控制决策。这个决策不是先精确到多少 kW，而是先输出“状态 + 目标/档位 + 原因”，用于和 IT/硬件侧对齐。

第一版输出：

```text
decision = {
  mode,
  target,
  reason_code
}
```

## 2. 输入/输出

### 2.1 输入

| 类别 | 参数 | 含义 |
|---|---|---|
| 云端半小时参数包 | `actual_ev_fee` | 实际 EV 充电费用 |
| 云端半小时参数包 | `actual_grid_cost` | 实际用电成本 |
| 云端半小时参数包 | `forecast_load` | 预期负荷/site load |
| 云端半小时参数包 | `forecast_ev_energy` | 预期 EV 充电量 |
| 云端半小时参数包 | `expected_soc` | 预期 SOC |
| 云端半小时参数包 | `valid_from / valid_to` | 参数有效期 |
| 本地实时状态 | `actual_soc` | 当前 BESS SOC |
| 本地实时状态 | `ev_request_power` | EV 当前请求功率/需求 |
| 本地实时状态 | `ev_actual_power` | EV 当前实际功率 |
| 本地实时状态 | `grid_import_power` | 当前并网点取电功率 |
| 本地实时状态 | `grid_import_available` | 电网剩余可用功率 |
| 本地实时状态 | `bms_status` | BMS 是否正常 |
| 本地实时状态 | `pcs_status` | PCS 是否正常/可控 |
| 本地实时状态 | `charger_status` | 充电桩状态 |
| 本地实时状态 | `timestamp` | 数据时间戳 |

### 2.2 输出

业务最小输出：

| 字段 | 含义 |
|---|---|
| `mode` | 当前控制状态 |
| `target` | 状态对应的目标/档位 |
| `reason_code` | 为什么做这个决策 |

建议给 IT 的输出格式：

```json
{
  "decision_id": "...",
  "as_of": "...",
  "mode": "BESS_DISCHARGE_TO_EV | BESS_CHARGE_TO_SOC | BESS_HOLD | EV_LIMIT | SAFE_PROTECT | SAFE_FALLBACK",
  "target": {
    "type": "SOC_BAND | EV_SUPPORT | HOLD | LIMIT",
    "value": "low | medium | high | target_soc_delta"
  },
  "reason_code": "EV_DEMAND_HIGH | SOC_LOW | SOC_IN_BAND | GRID_LIMIT | BESS_FAULT | DATA_STALE | CLOUD_EXPIRED | IMPORT_ONLY_LIMIT"
}
```

## 3. Constraints / 硬约束

这些约束必须先判断，不参与权重 trade-off：

| Constraint | 含义 | 影响 |
|---|---|---|
| `import_only = true` | M1 只允许从电网取电，不允许反送 | BESS 放电不能导致 export |
| `export_allowed = false` | 不允许卖电/反送 | 防止 BESS 放电超过现场负荷 |
| `SOC_min` | BESS 最低 SOC | 低于该值禁止放电 |
| `SOC_max` | BESS 最高 SOC | 高于该值禁止充电 |
| `SOC_low / SOC_high` | 云端/本地 SOC band | 判断 SOC 偏低、正常、偏高 |
| `P_grid_import_max` | 电网最大取电功率 | 超过则用 BESS 支撑或限制 EV |
| `P_pcs_charge_max` | PCS 最大充电功率 | 限制 BESS 最大充电动作 |
| `P_pcs_discharge_max` | PCS 最大放电功率 | 限制 BESS 最大放电动作 |
| `P_ev_gun_max` | 单枪最大功率 | 单枪不能超过上限 |
| `BMS_alarm` | 电池告警 | 触发 `SAFE_PROTECT` |
| `PCS_fault` | PCS 故障 | 触发 `SAFE_PROTECT` |
| `data_freshness` | 数据新鲜度 | 过期则 fallback/protect |
| `cloud_valid_to` | 云端参数有效期 | 过期则 `SAFE_FALLBACK` |
| `charge_discharge_mutex` | 充放电互斥 | BESS 不能同时充电和放电 |

约束优先级：

```text
安全/故障
> Import Only / 禁止反送
> 电网 import cap
> BESS SOC / PCS 边界
> EV 需求
> 云端 expected SOC
```

## 4. 状态空间、外部参数、动作空间

### 4.1 状态空间：现场现在是什么情况

```text
x_t = {
  SOC_actual,
  EV_power_request,
  EV_power_actual,
  grid_import_available,
  BMS_ok,
  PCS_ok,
  charger_ok,
  cloud_packet_fresh,
  protect_flag
}
```

### 4.2 云端外部参数：半小时参考

```text
theta_t = {
  actual_EV_charging_cost,
  actual_electricity_cost,
  forecast_load,
  forecast_EV_demand,
  SOC_reference
}
```

### 4.3 动作空间：本地希望系统做什么

```text
u_t = {
  BESS_mode,
  target,
  optional_EV_limit
}
```

其中：

```text
BESS_mode ∈ {
  BESS_CHARGE_TO_SOC,
  BESS_DISCHARGE_TO_EV,
  BESS_HOLD,
  EV_LIMIT,
  SAFE_PROTECT,
  SAFE_FALLBACK
}
```

`target` 可以先是档位，不一定是精确 kW：

```text
target ∈ {low, medium, high}
```

或者后续扩展为：

```text
target_soc_delta
target_power_kw
c_rate_level
```

## 5. 每个状态/动作的定义

| Mode | 主体视角 | 定义 | Target | 进入条件示例 | Reason code |
|---|---|---|---|---|---|
| `BESS_CHARGE_TO_SOC` | BESS | 给储能充电，让 SOC 往目标区间回升 | low/medium/high 或 target_soc_delta | SOC 低于 band，EV 无缺口或允许补 SOC | `SOC_LOW` |
| `BESS_DISCHARGE_TO_EV` | BESS | 储能向 EV 侧放电，减少 EV 缺口 | low/medium/high | EV 有缺口，SOC 高于下沿，BMS/PCS 正常 | `EV_DEMAND_HIGH` |
| `BESS_HOLD` | BESS | 储能待机，不充不放 | HOLD | SOC 在 band 内，无 EV 缺口 | `SOC_IN_BAND` |
| `EV_LIMIT` | EV/站点 | 限制 EV 充电能力 | low/medium/high | 电网不够，BESS 不可用或不应放电 | `GRID_LIMIT` |
| `SAFE_PROTECT` | BESS/PCS | 保护状态，停止相关充放电 | STOP | BMS 告警、PCS 故障、SOC 越界、关键数据异常 | `BESS_FAULT` |
| `SAFE_FALLBACK` | 本地控制 | 云端不可用时的保守运行 | DEFAULT | 云端参数过期、通信异常 | `CLOUD_EXPIRED` |

说明：

> `charge/discharge` 不是完整状态空间，而是动作空间。真正的状态空间是 SOC、EV 需求、电网余量、设备健康、数据新鲜度等现场状态。

## 6. 数据能否支持 M1

M1 V1 可以做的前提是至少拿到这些数据：

| 必须数据 | 来源 | 用途 |
|---|---|---|
| `SOC_actual` | BMS/EMS | 判断能不能充/放 |
| `BMS_ok / alarm` | BMS | 判断是否 protect |
| `PCS_ok / availability` | PCS | 判断是否能执行 |
| `EV_power_request / EV_power_actual` | 充电桩/OCPP/EMS | 判断 EV 需求和缺口 |
| `grid_import_power / grid_import_available` | PCC meter/电表 | 防止超过 import cap 和反送 |
| `timestamp` | 所有数据源 | 判断数据是否 fresh |

如果缺数据，降级原则：

| 缺失项 | 降级 |
|---|---|
| 无 PCC 电表 | 不做主动放电，避免反送风险 |
| 无 BMS SOC/告警 | 不自动充放电，进入 protect/idle |
| 无 PCS ACK/状态 | 停止新指令，待机或 protect |
| 无 EV 细分需求 | 用站级总负载/总桩功率做聚合控制 |
| 云端参数过期 | SAFE_FALLBACK |
| 数据超时 | fallback；关键安全数据超时则 protect |

## 7. 测试矩阵怎么来

测试不是随便想，而是从 constraints 和状态空间推导。

先切几个关键状态：

```text
SOC：low / normal / high / below_min / above_max
EV demand：none / normal / high_gap
Grid：headroom / at_cap / missing
BESS：available / limited / fault
Cloud：fresh / stale / missing
```

每条测试 case 都写：

```text
输入状态
触发的 constraint
expected mode
expected target
expected reason_code
```

示例：

| Case | 输入状态 | 触发点 | Expected mode | Reason |
|---|---|---|---|---|
| T1 | SOC high，EV high_gap，Grid at_cap，BESS available，Cloud fresh | EV 有缺口且 SOC 可放 | `BESS_DISCHARGE_TO_EV` | `EV_DEMAND_HIGH` |
| T2 | SOC low，EV high_gap，Grid at_cap，BESS available | SOC 低，不应放电 | `EV_LIMIT` | `SOC_LOW` / `GRID_LIMIT` |
| T3 | SOC low，EV none，Grid headroom，BESS available | 需要回 SOC | `BESS_CHARGE_TO_SOC` | `SOC_LOW` |
| T4 | SOC normal，EV none，Grid headroom | 无需动作 | `BESS_HOLD` | `SOC_IN_BAND` |
| T5 | BESS fault 或 BMS alarm | 安全优先 | `SAFE_PROTECT` | `BESS_FAULT` |
| T6 | Cloud stale | 云端参数过期 | `SAFE_FALLBACK` | `CLOUD_EXPIRED` |

