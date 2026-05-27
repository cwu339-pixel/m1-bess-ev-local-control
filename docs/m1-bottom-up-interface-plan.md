# M1 Bottom-up 本地接口与测试方案

日期：2026-05-26

用途：M1 bottom-up 技术附录，用于定义 IT 侧需要实现或对齐的参数、本地动作、接口和测试方案。

## 1. 最新结论

M1 范围：

```text
M1 = 储 + 充
BESS + EV
无 PV
Import Only
不考虑向电网卖电/反送
```

第一版不展开其他模型，也不做完整动态权重优化。

现在更重要的是：

```text
云端给半小时级时间序列参数包
其中包括实际 EV 充电费用、实际用电成本、预期负荷、预期 EV 充电量、预期 SOC
本地根据 M1 现场状态做简单动作
IT 只需要实现少数可调参数
设计测试场景验证这些参数是否有效
```

一句话：

> M1 bottom-up 工作包括：定义本地可调参数、参数影响的动作，以及云端模型、本地控制和 IT 数据接口的测试方案。

## 1.1 当前需要对齐的三样东西

当前重点不是大公式，而是形成产品、云端、IT 和硬件侧可以对齐的接口草案。

当前需要交付三样东西：

| 交付项 | 具体内容 | 主要受众 |
|---|---|---|
| 1. 问题定义 | M1 到底要解决什么：BESS + EV、Import Only、无 PV、无 export，本地根据云端半小时参数包和现场状态决定充/放/待机/限功率 | 云端 |
| 2. 参数表 | 云端下发哪些参数、IT 本地需要读哪些参数、本地输出哪些动作 | 云端/IT |
| 3. 测试方案 | 不同 SOC band、EV 需求、电网限制、BMS 告警、guidance 过期时，本地应该怎么动作 | IT |

一句话：

> 先把 M1 的“问题、参数、测试”定义出来，方便对齐接口怎么接、本地控制怎么测。

Non-goals:

- other control models；
- 完整目标函数；
- 动态权重；
- economic arbitrage；
- PV/export/V2G。

当前重点:

```text
M1 里，IT 能收到哪些参数；
这些参数在不同 SOC band 下让本地做什么动作；
如何测试这些动作是否符合预期。
```

## 2. M1 里本地到底要做什么

M1 只有 BESS + EV，没有 PV，也不考虑 export。

第一版先把输入/输出说简单：

```text
输入：
云端给的半小时粒度参数包
+ 本地实时状态
+ 本地 constraints

输出：
一个具体动作状态 + 对应参数
例如 charge(level), discharge(level), idle, limit(level), protect, fallback
```

这里不能只输出状态名字。硬件端最终需要知道“这个状态具体要做什么”，所以第一版至少要把状态和参数说清楚，比如 charge 是给 BESS 充电，discharge 是 BESS 放电给 EV，level 表示充放电强度档位或目标 SOC 增量。

主体统一用 BESS 视角。也就是说：

```text
charge = 给储能电池充电
discharge = 储能电池向 EV/充电桩放电
```

本地主要状态只有几类：

| 状态 | 含义 | 是否需要参数 |
|---|---|
| `charge(level)` | 给 BESS 充电 | 需要，表示充电强度/目标 |
| `discharge(level)` | BESS 放电支援 EV | 需要，表示放电强度/目标 |
| `idle` | BESS 待机，不充不放 | 不需要 |
| `limit(level)` | 限制 EV 充电功率 | 需要，表示限功率强度 |
| `protect` | BMS/PCS 告警等保护动作 | 不需要，直接保护 |
| `fallback` | 云端参数过期/通信异常，本地保守运行 | 可选，使用默认保守参数 |

## 3. M1 不做什么

第一版 M1 不做：

- 不考虑 PV 消纳；
- 不考虑卖电/反送电；
- 不考虑 V2G；
- 不做other control models；
- 不做完整动态权重优化；
- 不把economic arbitrage放到本地核心动作里；
- 不让 LLM 进入实时控制闭环。

## 4. 云端下发参数：先分清“云端已有数据”和“IT 控制参数”

云端参数不是只有 SOC guidance，而是一组半小时粒度时间序列：

| 云端已有参数 | 含义 | 在 M1 里的作用 |
|---|---|---|
| 实际 EV 充电费用 | EV 充电收入/费用 | 主要用于收益复盘和云端模型评估 |
| 实际用电成本 | 场站买电成本 | 主要用于云端经济调度和复盘 |
| 预期负荷 | 未来半小时 site load 预测 | 判断未来是否有电网压力 |
| 预期 EV 充电量 | 未来半小时 EV 需求预测 | 判断未来是否需要保留/补充 SOC |
| 预期 SOC | 云端希望 SOC 怎么走 | 本地最直接参考的 SOC 目标/走廊 |

所以现在要分两层：

```text
云端参数包：这些数据都可以下发或共享
IT 控制参数：先只挑 2-3 个真正影响本地动作的参数
```

给 IT 的控制参数应保持少量，先定义清楚 IT 端要接什么、能调什么。

### 推荐核心参数

| 参数 | 建议字段 | 说明 | 作用 |
|---|---|---|---|
| 预期 SOC / SOC 区间 | `expected_soc`, `soc_p10`, `soc_p50`, `soc_p90` 或 `soc_low`, `soc_target`, `soc_high` | 半小时粒度 | 决定当前 SOC 是偏低、正常、偏高 |
| 充电/放电强度档位 | `c_rate_level` 或 `power_gain` | 可以是 0/低/中/高 | 决定 BESS 充电或放电时用多大功率 |
| 有效期/模式 | `valid_to`, `mode=M1`, `fallback_policy` | 防止过期 guidance 被继续使用 | 过期后进入本地保守策略 |

如果只能保留两个：

```text
1. SOC 区间/目标
2. C-rate/功率档位
```

有效期可以作为接口基础字段，不一定算“控制参数”。

## 5. SOC 分区和本地动作

M1 可以先用 SOC 分区做 Zone-Gain-Routing。

```text
Zone 1：SOC 低于下沿
Zone 2：SOC 在区间内
Zone 3：SOC 高于上沿
```

示例：

| SOC 状态 | 本地倾向 | BESS 动作 |
|---|---|---|
| SOC < soc_low | 优先补 SOC | 尽量不放电；允许时从电网给 BESS 充电 |
| soc_low <= SOC <= soc_high | 正常运行 | 有 EV 缺口时可适度放电；无缺口时待机 |
| SOC > soc_high | 可以释放一点 SOC | 有 EV 缺口时更积极放电支援 EV |

If cloud provides p10/p50/p90, interpret them as:

```text
p50 = 云端最希望的 SOC 中心
p10/p90 = 允许本地模糊调度的走廊
```

本地不需要精确追一个点，而是在走廊里做动作。

## 6. C-rate/功率档位怎么理解

C-rate 不需要讲得太复杂，可以先理解成“电池充放电速度档位”。

例如：

| 档位 | 含义 | 可能动作 |
|---|---|---|
| 0 | 不动作 | BESS 待机 |
| 低 | 小功率 | 轻微充电/轻微放电 |
| 中 | 中功率 | 常规补功率 |
| 高 | 高功率 | SOC 偏离大或 EV 缺口大时使用 |

第一版不用精确说每档是多少 kW，可以先让 IT 支持参数化：

```text
c_rate_level = 0 / low / medium / high
```

后续再映射成实际功率：

```text
P_bess_cmd = f(c_rate_level, PCS_limit, SOC_limit, BMS_limit)
```

## 7. M1 本地策略规则草案

## 7.0 中间数学表达

第一版输出不是简单 0/1，而是“状态 + 参数”。可以写成：

```text
a_t = (mode_t, target_t, reason_t)

mode_t ∈ {charge, discharge, idle, limit, protect, fallback}
```

其中：

```text
mode_t   = 当前动作状态
target_t = 这个状态对应的目标/强度
reason_t = 为什么进入这个状态
```

`target_t` 第一版不一定直接是 kW，可以先是档位或 SOC 目标：

```text
target_t ∈ {0, low, medium, high}
```

或者：

```text
target_t = delta_soc_target
```

例子：

```text
charge(medium, reason = SOC_below_band)
discharge(high, reason = EV_gap_and_SOC_high)
idle(reason = SOC_in_band_no_EV_gap)
limit(medium, reason = grid_limit_and_BESS_unavailable)
protect(reason = BMS_alarm)
fallback(reason = cloud_expired)
```

### 输入

云端半小时参数包：

```text
C_t = {
  EV_fee_t,
  grid_cost_t,
  forecast_load_t,
  forecast_ev_energy_t,
  expected_soc_t
}
```

本地实时状态：

```text
X_t = {
  SOC_t,
  EV_request_t,
  EV_actual_t,
  Grid_available_t,
  BMS_status_t,
  PCS_status_t
}
```

本地硬约束：

```text
K = {
  SOC_min,
  SOC_max,
  P_grid_import_max,
  P_pcs_charge_max,
  P_pcs_discharge_max,
  P_ev_gun_max,
  valid_to
}
```

### 关键中间变量

```text
SOC_error_t = SOC_t - expected_soc_t

EV_gap_t = max(0, EV_request_t - Grid_available_t)

cloud_valid_t = now <= valid_to
```

SOC 分区：

```text
SOC_zone_t =
  low      if SOC_t < soc_low
  normal   if soc_low <= SOC_t <= soc_high
  high     if SOC_t > soc_high
```

### 硬约束先判断

```text
if BMS_status_t is alarm or PCS_status_t is fault:
    a_t = protect

else if cloud_valid_t = false:
    a_t = fallback

else if SOC_t <= SOC_min:
    discharge is not feasible

else if SOC_t >= SOC_max:
    charge is not feasible
```

### 状态选择

在可行状态集合 `A_feasible(t)` 里面选一个状态：

```text
a_t = argmin J_t(a),  a ∈ A_feasible(t)
```

`J_t(a)` 可以先理解成综合不满意分：

```text
J_t(a) =
  w_ev   * EV_gap_after(a)
+ w_soc  * |SOC_after(a) - expected_soc_t|
+ w_grid * Grid_risk_after(a)
+ w_bess * BESS_stress_after(a)
+ w_sw   * Switch_penalty(a, a_{t-1})
```

第一版不需要把所有项都精确算出来，可以先用规则近似：

```text
if EV_gap_t > 0 and SOC_zone_t = high:
    a_t = discharge(high or medium)

if EV_gap_t > 0 and SOC_zone_t = low:
    a_t = limit(medium or high)

if EV_gap_t = 0 and SOC_zone_t = low:
    a_t = charge(low or medium)

if EV_gap_t = 0 and SOC_zone_t = normal:
    a_t = idle
```

一句话：

> 云端参数和本地状态进入公式，先过硬约束，再根据 SOC_zone 和 EV_gap 选择状态和参数；目标是减少 EV 缺口、减少 SOC 偏离，同时不违反电网、电池和设备限制。

## 7.1 状态空间定义

这张表是给 IT/硬件侧对齐用的：每个状态到底是什么意思、需要什么参数、什么时候进入。

| 状态 | 主体视角 | 状态定义 | 参数 | 进入条件示例 | 输出解释 |
|---|---|---|---|---|---|
| `charge(level)` | BESS | 给储能电池充电 | `level = low/medium/high` 或 `delta_soc_target` | SOC 低于预期 SOC band，且无 EV 缺口或电网允许 | 希望 BESS 往目标 SOC 回升 |
| `discharge(level)` | BESS | 储能电池向 EV/充电桩放电 | `level = low/medium/high` | EV 有功率缺口，且 SOC 高于下沿，BMS/PCS 允许 | 用 BESS 支援 EV，减少 EV 缺口 |
| `idle` | BESS | 储能不充不放 | 无 | SOC 在 band 内，EV 无缺口，或没有必要动作 | 保持当前状态，避免无意义充放电 |
| `limit(level)` | EV/站点 | 限制 EV 充电能力 | `level = low/medium/high` | EV 有需求但电网不够，且 BESS 不能放或不应放 | 降低 EV 侧需求，避免电网/SOC 越界 |
| `protect` | BESS/PCS | 保护状态，停止相关充放电 | 无 | BMS 告警、PCS 故障、SOC 越界、关键数据异常 | 安全优先，停止 BESS 动作 |
| `fallback` | 本地控制 | 云端不可用时的保守策略 | 可选默认参数 | 云端参数过期、通信异常 | 不再按云端参数主动动作，只做本地安全策略 |

`level` 第一版可以先不映射成具体 kW，只定义为档位：

| level | 含义 | 后续可映射 |
|---|---|---|
| `low` | 小幅充/放/限 | 低 C-rate 或小功率 |
| `medium` | 常规充/放/限 | 中 C-rate 或常规功率 |
| `high` | 较强充/放/限 | 高 C-rate 或接近 PCS 可用上限 |

如果硬件侧说不能执行 “充 5% SOC” 这种目标，可以退化成：

```text
charge(low/medium/high)
discharge(low/medium/high)
```

也就是先用档位反复执行，直到 SOC 回到目标区间或状态变化。

### Step 1：先检查硬约束

```text
如果 BMS/PCS 告警：
  停止相关 BESS 充放电，进入保护

如果 SOC <= SOC_min：
  禁止 BESS 放电

如果 SOC >= SOC_max：
  禁止 BESS 充电

如果 cloud guidance 过期：
  进入 fallback，本地保守运行
```

### Step 2：判断 EV 缺口

```text
EV_request = 车辆当前合理请求功率
Grid_available = 电网当前可用功率
EV_gap = max(0, EV_request - Grid_available)
```

### Step 3：根据 SOC zone 决定 BESS 动作

```text
如果 EV_gap > 0 且 SOC 高于下沿：
  BESS 可以放电补 EV_gap

如果 EV_gap > 0 但 SOC 低于下沿：
  BESS 少放或不放，EV 降功率

如果 EV_gap = 0 且 SOC 低于目标/下沿：
  可以从电网给 BESS 充电

如果 SOC 在区间内且没有 EV 缺口：
  BESS 待机
```

### Step 4：输出动作和原因

本地输出：

- `mode = charge / discharge / idle / limit / protect / fallback`
- `target = low / medium / high` 或 `delta_soc_target`
- `reason_code`
- `local_mode = normal / fallback / protect`

后续扩展时再输出：

- `P_ev_cmd`
- `P_bess_cmd`
- `c_rate_level`

## 8. IT 需要支持的输入/输出

### IT/本地需要读到的输入

| 输入 | 说明 |
|---|---|
| 当前 SOC | 电池实时 SOC |
| BMS/PCS 状态 | 是否允许充放电 |
| PCS 最大充放电功率 | 当前可执行功率上限 |
| EV 请求功率 | 车辆/桩当前请求 |
| EV 实际功率 | 实际输出 |
| 电网可用功率/站点上限 | 防止超限 |
| 云端半小时参数包 | 实际 EV 充电费用、实际用电成本、预期负荷、预期 EV 充电量、预期 SOC |
| 预期 SOC / SOC 区间 | 半小时 SOC 区间/目标，作为本地动作主要参考 |
| guidance 有效期 | 防止过期 |

### 本地需要输出/回传

| 输出 | 说明 |
|---|---|
| `action` | `charge / discharge / idle / limit / protect / fallback` |
| `reason_code` | 为什么做这个动作 |
| `local_mode` | normal / fallback / protect |
| `soc_error` | 当前 SOC 和云端目标差多少 |
| `ev_gap` | EV 充电缺口 |
| `override_reason` | 为什么没有跟随云端或没有给满 EV |

第一版可以先不输出：

| 后续字段 | 说明 |
|---|---|
| `P_ev_cmd` | 给充电桩的具体功率 |
| `P_bess_cmd` | 给 BESS/PCS 的具体功率 |
| `c_rate_level` | 充放电速度档位 |

## 9. M1 测试场景

第一版测试不要铺太开，先测参数是否真的能影响动作。

| 测试 | 输入 | 预期动作 | 验证点 |
|---|---|---|---|
| T1：SOC 充足，车有缺口 | SOC 高于区间，EV 需要 150kW，电网只能 100kW | BESS 放电补一部分或全部缺口 | EV 缺口减少 |
| T2：SOC 偏低，车有缺口 | SOC 低于区间，EV 需要 150kW，电网只能 100kW | BESS 少放或不放，EV 降功率 | SOC 保护优先生效 |
| T3：无 EV 缺口，SOC 偏低 | EV 需求低或无车，SOC 低于区间 | 从电网给 BESS 充电 | SOC 往 guidance 回归 |
| T4：SOC 在区间内，无缺口 | SOC 在走廊内，EV 能由电网满足 | BESS 待机 | 避免无意义充放 |
| T5：BMS/PCS 告警 | 任意 SOC，触发告警 | 停止 BESS 相关动作 | 硬约束生效 |
| T6：guidance 过期 | 云端 valid_to 过期 | 进入 fallback | 不继续按旧策略 |

## 10. 当前最应该交付的东西

1. M1 模式定义；
2. 云端下发参数表；
3. 本地输入/输出字段；
4. SOC zone 到动作的规则；
5. IT 可调参数；
6. 6 个测试场景；
7. 每个场景的预期输出和 reason code。
