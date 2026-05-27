# 会议纪要：Import Only 参数化与测试框架

日期：2026-05-27

主题：充电桩充电策略、本地控制接口、测试环境与模型运维框架对齐。

## 1. 会议核心结论

本次会议围绕充电桩充电策略制定展开。当前阶段先不展开所有控制模型，而是聚焦最简单的 M1 / Import Only 场景，先把问题定义、约束条件、参数列表和测试方式讲清楚。

核心方向：

```text
云端输出半小时参数包；
本地根据实时状态和硬约束；
输出少数几个可解释动作；
再通过测试环境验证这些动作是否符合预期。
```

## 2. 充电策略的考虑维度

### 硬约束

先考虑已经设定好的边界条件，包括：

- SOC 上下限；
- 单枪功率上限；
- PCS 功率上限；
- 电网 import 上限；
- BMS / PCS / charger 告警；
- 云端参数有效期；
- 充放电互斥；
- Import Only 下不允许反送。

这些条件不是优化目标，而是动作过滤器。违反硬约束的动作不能进入后续选择。

### 用户体验

本地策略需要考虑车主和站点运营希望得到什么结果，包括：

- 单车充电速度；
- 充电缺口；
- 完成时间；
- 等待时间；
- 多车同时充电时的公平分配；
- 站点整体吞吐。

当前第一版可以先把“尽量减少 EV 充电缺口”作为主要用户体验目标。

### 云端 SOC 匹配

本地 SOC 需要尽量跟随云端预期，但不是机械照抄。云端给的是半小时粒度参考，本地需要根据现场 EV 需求、电网能力、BESS 状态和安全约束做实时调整。

## 3. 参数化与 IT 接口

Ning 强调当前要把策略参数化，而且参数数量不要太多，先控制在两三个真正影响动作的参数，方便和 IT 部门对接。

第一版可以优先定义：

| 参数 | 作用 |
|---|---|
| `soc_low / soc_target / soc_high` | 判断 SOC 偏低、正常、偏高 |
| `c_rate_level` 或 `power_gain` | 决定充放电强度档位 |
| `valid_to / fallback_policy` | 判断云端参数是否过期，过期后如何 fallback |

IT 部门需要知道：

```text
收到这些参数后；
本地控制器能进入哪些 mode；
每个 mode 对应什么动作；
什么时候进入 fallback/protect；
动作和 reason_code 如何回传。
```

## 4. 当前只考虑 M1 / Import Only

当前范围：

```text
M1 = BESS + EV
Import Only
无 PV
无 export
无 V2G
```

因此当前不要讨论卖电、光伏消纳、V2G 或其他复杂模型。

本地动作先定义为：

```text
decision = {
  mode,
  target,
  reason_code
}
```

建议 mode：

- `BESS_CHARGE_TO_SOC`
- `BESS_DISCHARGE_TO_EV`
- `BESS_HOLD`
- `EV_LIMIT`
- `SAFE_PROTECT`
- `SAFE_FALLBACK`

## 5. 测试环境与迭代框架

测试环境需要搭建在本侧，并和云端宏观模型整合。IT 部门主要协助提供底层数据和本地控制执行能力。

测试不是随便造场景，而是从状态空间和 constraints 推导：

```text
SOC：low / normal / high / below_min / above_max
EV demand：none / normal / high_gap
Grid：headroom / at_cap / missing
BESS：available / limited / fault
Cloud：fresh / stale / missing
```

每个测试 case 应该包含：

```text
输入状态；
触发约束；
expected mode；
expected target；
expected reason_code。
```

## 6. Perfect / MLflow 框架补充

会议还讨论了模型运维框架：

- Perfect / Prefect：用于编排数据 workflow，可替代手工定时任务或 Airflow 类流程；
- MLflow：用于模型层面的运维、记录、评估和比较；
- 后续可以叠加 MCP / LLM 层，用于参数检查、模型选择和辅助调参。

当前阶段不需要把这部分和 M1 本地控制混在一起实现。它更像后续模型迭代和自动化运维框架。

## 7. 后续任务

### 本地控制侧

针对 Import Only 场景，尽快输出：

1. 问题定义；
2. 约束条件；
3. 参数列表；
4. mode / target / reason_code；
5. 基础测试场景。

### 云端/模型侧

继续完善模型框架、参数表和测试环境，并研究 Perfect / MLflow 工作流。

### 对齐方式

先完成 M1 的接口草案，再组织产品、云端、IT 和硬件侧快速对齐。当前重点不是证明算法最优，而是让接口、参数和测试框架先能跑通。
