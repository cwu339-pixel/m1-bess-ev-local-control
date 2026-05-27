# M1 本地控制接口整理

日期：2026-05-26

用途：先对齐 M1 的问题定义、约束条件和参数列表。当前版本只考虑 M1，不展开完整优化算法。

## 外部调研后的收敛结论

参考 OCPP Smart Charging、OpenEMS Edge、EVerest EnergyManager 和高功率充电站分层控制相关资料后，M1 第一版建议按三层理解：

| 层级 | 时间颗粒度 | 负责什么 | 当前需要定义什么 |
|---|---|---|---|
| 云端计划层 | 半小时 | 费用、成本、预期负荷、预期 EV 充电量、预期 SOC | 云端半小时参数包 |
| 本地 EMS 决策层 | 秒级/短周期，具体取决于设备数据刷新 | 根据本地实时状态和约束，选择动作状态 | `mode + target + reason_code` |
| 硬件保护层 | 设备内部实时保护 | BMS/PCS/充电桩安全保护 | 不由本接口定义，只作为 constraints |

所以当前不是写完整算法，而是定义：

```text
云端半小时参数包 + 本地实时状态
        ↓
constraints 过滤不可行动作
        ↓
本地输出 mode + target + reason_code
```

## 一句话问题定义

> 在 M1（储能 + 充电桩，Import Only）场景下，云端给半小时参数包，本地根据实时状态和硬约束，判断储能系统当前应该充电、放电支援 EV、待机、限制 EV、保护，还是进入 fallback。

第一版输出不是具体 kW，而是一个可解释的控制决策：

```text
decision = mode + target + reason_code
```

其中：

| 字段 | 含义 |
|---|---|
| `mode` | 本地要进入哪种控制状态 |
| `target` | 这个状态的目标或强度，先用 low/medium/high |
| `reason_code` | 为什么进入这个状态，方便测试和回传 |

## 表 1：问题定义

| 项目 | 内容 |
|---|---|
| 模式 | M1 |
| 场景 | 储能 + 充电桩 |
| 电网方向 | Import Only |
| 不考虑 | PV、export、V2G |
| 输入 | 云端半小时参数包 + 本地实时状态 |
| 约束 | constraints 作为动作过滤器 |
| 输出 | `mode + target + reason_code` |
| 第一版目标 | 先定义接口、状态空间和动作空间，不做精确 kW dispatch |

核心逻辑：

```text
云端参数 + 本地实时状态
        ↓
形成当前状态空间
        ↓
constraints 过滤掉不可行动作
        ↓
输出 mode + target + reason_code
```

## 表 2：约束条件

| 参数 | 含义 | 本地动作影响 |
|---|---|---|
| `SOC_min` | 电池最低 SOC | 低于它禁止放电 |
| `SOC_max` | 电池最高 SOC | 高于它禁止充电 |
| `SOC_low / SOC_high` | 云端 SOC band | 判断 SOC 偏低、正常、偏高 |
| `P_grid_import_max` | 电网最大取电功率 | 超过就限功率或用电池补 |
| `P_pcs_charge_max` | PCS 最大充电功率 | 限制电池最大充电能力 |
| `P_pcs_discharge_max` | PCS 最大放电功率 | 限制电池最大放电能力 |
| `P_ev_gun_max` | 单枪最大功率 | 单车不能超过，比如 160kW |
| `BMS_alarm` | 电池告警 | 告警则 `SAFE_PROTECT` |
| `PCS_fault` | PCS 故障 | 故障则 `SAFE_PROTECT` |
| `cloud_valid_to` | 云端参数有效期 | 过期则 `SAFE_FALLBACK` |
| `charge_discharge_mutex` | 充放电互斥 | 电池不能同时充和放 |

## 表 3：参数列表

| 类别 | 参数 | 含义 |
|---|---|---|
| 云端参数 | `actual_ev_fee` | 实际 EV 充电费用 |
| 云端参数 | `actual_grid_cost` | 实际用电成本 |
| 云端参数 | `forecast_load` | 预期负荷 |
| 云端参数 | `forecast_ev_energy` | 预期 EV 充电量 |
| 云端参数 | `expected_soc` | 预期 SOC |
| 云端参数 | `valid_from / valid_to` | 参数有效期 |
| 本地实时状态 | `actual_soc` | 当前 SOC |
| 本地实时状态 | `ev_request_power` | EV 请求功率 |
| 本地实时状态 | `ev_actual_power` | EV 实际功率 |
| 本地实时状态 | `grid_available_power` | 电网可用功率 |
| 本地实时状态 | `bms_status` | BMS 状态 |
| 本地实时状态 | `pcs_status` | PCS 状态 |
| 本地实时状态 | `charger_status` | 充电桩状态 |
| 本地输出 | `mode` | `BESS_CHARGE_TO_SOC / BESS_DISCHARGE_TO_EV / BESS_HOLD / EV_LIMIT / SAFE_PROTECT / SAFE_FALLBACK` |
| 本地输出 | `target` | `low / medium / high`，后续可映射成具体功率或 SOC 目标 |
| 本地输出 | `reason_code` | 为什么做这个动作，见下方枚举 |
| 本地输出 | `local_mode` | `normal / fallback / protect` |
| 本地输出 | `soc_error` | SOC 偏差 |
| 本地输出 | `ev_gap` | EV 充电缺口 |
| 本地输出 | `override_reason` | 为什么覆盖云端建议 |

## Mode 枚举

| Mode | 含义 |
|---|---|
| `BESS_CHARGE_TO_SOC` | 给储能充电，让 SOC 往预期区间回升 |
| `BESS_DISCHARGE_TO_EV` | 储能放电支援 EV，减少 EV 充电缺口 |
| `BESS_HOLD` | 储能待机，不充不放 |
| `EV_LIMIT` | 限制 EV 充电功率，避免电网/SOC/设备越界 |
| `SAFE_PROTECT` | 保护状态，停止相关充放电 |
| `SAFE_FALLBACK` | 云端参数过期或通信异常时，本地保守运行 |

## Reason Code 枚举

| Reason code | 含义 |
|---|---|
| `SOC_LOW` | SOC 低于目标区间或接近下限 |
| `SOC_HIGH` | SOC 高于目标区间 |
| `SOC_IN_BAND` | SOC 在目标区间内 |
| `EV_DEMAND_HIGH` | EV 需求较高，电网单独无法满足 |
| `GRID_LIMIT` | 电网取电接近或达到上限 |
| `BESS_FAULT` | BMS/PCS 告警或故障 |
| `DATA_STALE` | 本地关键数据过期 |
| `CLOUD_EXPIRED` | 云端参数包过期 |
| `IMPORT_ONLY_LIMIT` | M1 不允许反送电，动作受限 |

## 简短说明

当前版本先不讨论完整最优算法，只定义 M1 接口和动作空间。

```text
输入形成当前状态；
constraints 过滤不可行动作；
最后输出 mode + target + reason_code。
```
