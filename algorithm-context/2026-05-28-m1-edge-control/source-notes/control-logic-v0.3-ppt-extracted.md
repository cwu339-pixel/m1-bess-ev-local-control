# 控制逻辑_技术交流_v0.3 PPT 提取文本

来源文件: `/Users/wuchenghan/Library/Containers/com.tencent.WeWorkMac/Data/Documents/Profiles/C80E554D3DBA0960EF7ADFAB485D6BEB/Caches/Files/2026-05/0122c3c2595d49f4dc1a4dc7cea06cfa/控制逻辑_技术交流_v0.3.pptx`

总页数: 21

> 由 PPTX XML 提取，用于 M1 云端-本地接口整理。

## Slide 1
- 直流母线 光储充 控制系统
- 技术交流  ·  v0.3
- 基于 README v0.3  ·  2026-05-13  ·  v0.1 (mode 真值表) → v0.2 (A/B/C 路由) → v0.3 (孤岛 / 灰度 / 合规)
- 本地为主、SOC 为引导
- EMS 是傻执行器
- 云做思考，边缘做反应
- DC Bus PV + BESS + EV Fast Charging — Edge-First, Cloud-Advisory

Notes:
- 1

## Slide 2
- 议程
- v0.3 · 技术交流
- 01
- 业务定位与本版本范围
- 光储能 → 光储充；EMS 做什么 / 不做什么
- 02
- 新控制逻辑
- A/B/C/D 优先级 + 三档时间常数
- 03
- 云-边契约
- advisory schema + SOC band + 灰度上线
- 04
- 状态机 + 可观测性
- 5 系统态 + 日志/KPI/Shadow delta
- 05
- BusMan 代码迁移路线
- 6 Steps · 约 3–4 个月
- 2 / 22

Notes:
- 2

## Slide 3
- 01
- 业务定位
- 公共直流快充站 · 光储充一体化（光储能 → 光储充）
- v0.3 · 技术交流
- 硬件容量与场景参数
- BESS
- 250 kWh
- 锂电池组 + BMS
- PCS
- 30 / 60 kW
- 双向（充/放）
- EV 单枪
- 240 kW DC
- CCS/GBT，N 把
- PV
- 容量 TBD
- DC 母线直挂
- 与上一代 telcoman 的关系
- 沿用：
- Modbus 寄存器映射、双进程架构（busman 采集 + controller 决策）、Redis 总线、ThingsBoard MQTT
- 重做：
- 状态机（7 mode 真值表 → 6 系统态 + A/B/C 路由）、云-边接口（mode 下发 → SOC band advisory + weight 旋钮）
- 删除：
- 本地优化栈（cvxpy LP + ARIMAX + XGBoost 约 1400 行）、dispatch_bridge（374 行）；预测和优化全部上云
- 新增：
- EV 桩接入（OCPP + Modbus 0x0900-09FF）、SOC 概率走廊 p10/p50/p90、advisory_weight 灰度旋钮、SOFT/HARD HOLD + ISLAND_AC 兜底、动态 MIC margin、shadow delta 上报
- 3 / 22

Notes:
- 3

## Slide 4
- 01a
- 本版本范围（Scope）
- EMS 越简单越好：bug 表面积小、调试可预测、7×24 可维护
- v0.3 · 技术交流
- ✓  本地 EMS 做
- ✓
- Follow
- 按 SOC band 引导（不精确跟踪）
- ✓
- Routing
- 按 A/B/C 优先级路由功率
- ✓
- Limit
- 在 D 限制（MIC/BMS/物理边界）内执行
- ✓
- Telemetry
- 上行状态、D 触发日志、Shadow delta
- ✗  本地 EMS 不做（在别处）
- ✗
- 预测 / 优化
- 云端：LP/QP/ML 模型
- ✗
- 经济性建模
- 云端：套利收益、IRR
- ✗
- EV 公平分配
- 桩端：OCPP ChargingProfile
- ✗
- BESS SOC 估计 / 均衡
- BMS 固件
- ✗
- PCS 电压电流环 + DC 母线电压控制
- PCS 厂商：既是 AC↔DC 接口，也是 DC 母线电压源
- OCPP 边界：
- EMS 只下发 P_gun_pool_max 总盘子；per-枪分配由 OCPP 站控 (SiteSupervisor) 负责，EMS 不感知 per-gun 细节。
- 原则：
- 任何把『思考』或『优化』塞回 EMS 的需求，默认拒绝。
- 4 / 22

Notes:
- 4

## Slide 5
- 01b
- DC 母线拓扑（提示）
- PCS 单点接网 · DC 母线作为枢纽 · 双向 PCS 既是 AC↔DC 接口，也是 DC 母线电压源
- v0.3 · 技术交流
- AC 电网
- PoC 电表
- 0x4022
- 双向 PCS
- 30/60 kW
- 站内 AC 负荷
- AC 配电
- DC 母线 750–1000 V
- PV DC/DC
- PV 阵列
- Bi-DC/DC
- BESS 250kWh
- EV #1
- 240 kW
- EV #2
- 240 kW
- EV #3
- 240 kW
- telcoman 边缘 EMS
- 本地为主 · SOC 引导
- L1 200ms · L2 1s · L3 30min
- 云端
- advisory 30min
- 5 / 22

Notes:
- 5

## Slide 6
- 02
- 新控制逻辑总览
- 本地为主、SOC 为引导：A/B/C 路由 + D 兜底 + 三档时间常数
- v0.3 · 技术交流
- 30 min
- L3 · 云端 advisory
- 远程，引导
- 云端消化 4 类预测（day-ahead）
- 下发 SOC band（P10/P50/P90）
- 下发 allow_sell / allow_buy_grid
- TTL 过期 → 切默认保守
- 1 s
- L2 · 协调环
- 本地自主，主决策
- C 列表：PV 利用（车→负荷→网）
- A 列表：BESS 放电（车+负荷→卖电）
- B 列表：BESS 充电（光伏→电网）
- D-1 负荷优先 / D-5 桩故障 校验
- 套利 A-3/B-2 由 advisory 解锁
- 200 ms
- L1 · D 快环
- 本地，硬约束
- D-2 MIC 守门（反馈式）
- D-3 BESS 物理边界
- D-4 BMS 告警兜底
- L1' 事件中断：BMS 一级告警
- 过限就剪，不参与策略
- 6 / 22

Notes:
- 6

## Slide 7
- 02a
- A) BESS 放电优先级
- ≤ 60 kW · 受 SOC 下限保护 · 每协调环重新评估
- v0.3 · 技术交流
- 1
- 支援 EV 充电（聚合 AC 负荷）
- BESS 看到的是 (EV + AC 负荷) 合并净需求；按最大放电 60 kW 出力
- 本地自主
- 2
- 抵扣站内 AC 负荷
- 无 EV 充电时仍放电填负荷，减少买电
- 本地自主
- 3
- 卖电
- 前两项满足且 SOC 偏高 + MIC_export 余量 + allow_sell = true
- 需云解锁
- 工程要点：
- A 列表每协调周期重新评估 — 任何时刻 EV 进站立刻把 BESS 从 A-3 卖电召回 A-1。
- BESS 看到的就是合并的净需求；具体如何被 EV/AC负荷瓜分由 DC 母线自然完成，EMS 不区分。
- 7 / 22

Notes:
- 7

## Slide 8
- 02b
- B) BESS 充电优先级
- ≤ 30 kW · 受 SOC 上限保护
- v0.3 · 技术交流
- 1
- 光伏（PV 余量）
- PV 出力 > 负荷 + 充电 → 余量充 BESS。本地必然执行
- 本地自主
- 2
- 电网
- SOC < SOC*(t) band 下限 + advisory.allow_buy_grid = true（如谷电窗口）
- 需云解锁
- 为什么 B 列表只有 2 项？
- BESS 充电的电只有两个来源：自家发的（光伏）和外面买的（电网）。
- B-1 PV 余量是免费的，本地见到余量就吃，不需要云授权。
- B-2 电网充要花钱、要占 MIC 进网容量，必须云端确认（谷电 + SOC 偏低）才划算。
- 云 TTL 过期 → B-2 视为禁用，本地只跑 B-1。
- 8 / 22

Notes:
- 8

## Slide 9
- 02c
- C) PV 利用优先级
- PV 出力按 给车 → 给负荷 → 给电网 自上而下消化
- v0.3 · 技术交流
- 1
- 给车（直供 EV 充电）
- DC 母线最短路径，效率最高；优先满足在站 EV 请求
- 本地自主
- 2
- 给负荷（直供站内 AC）
- 经 PCS 反向，给站内辅助设备（照明 / HVAC / 安防）
- 本地自主
- 3
- 给电网（卖电）
- 前两项消化不完的余量 → 卖电（受 MIC_export 限 + allow_sell）
- 需云解锁
- 兜底（不列入 C，自动发生）：
- 前 3 项消化不完的 PV 余量 → 自动经 B-1 进 BESS；BESS 满后 PV DC/DC 退 MPPT 限发。
- 9 / 22

Notes:
- 9

## Slide 10
- 02d
- D) 限制约束（全局硬约束）
- 任何 A/B/C 动作不可突破；分级守底 200 ms / 1 s / 事件
- v0.3 · 技术交流
- 代号
- 名称
- 规则
- 评估周期
- D-1
- 负荷优先
- PV/BESS 放电必须先满足 (AC 负荷 + 已握手 EV)；未满足时禁止卖电/出网；无法满足 → SAFE_HOLD
- 1 s 协调环
- D-2
- MIC 守门
- 以 PoC 电表 (0x4022) 为准，反馈式：overshoot = P_PoC − (MIC − margin)；margin 动态
- 200 ms 快环
- D-3
- BESS 物理边界
- SOC ∈ [hard_min, hard_max]；单体电压、温度；越限立即停 BESS
- 200 ms 快环
- D-4
- BMS 一级告警
- 0x0764 事件驱动 + 200 ms 兜底；触发 → 停 BESS + HARD_HOLD
- 事件 + 200 ms
- D-5
- 桩端故障
- 单枪退出，其余继续；多桩同时故障 → DEGRADED
- 1 s 协调环
- 10 / 22

Notes:
- 10

## Slide 11
- 03
- 云-边契约（advisory v0.3）
- SOC 概率走廊 + allow_* 解锁位 + advisory_weight 灰度旋钮
- v0.3 · 技术交流
- advisory.json
- {
- "schema_version": "0.3",
- "issued_at": "2026-05-12T08:00:00Z",
- "horizon_hours": 24,
- "tick_minutes": 30,
- "soc_band": [
- {"t_offset_min":0,
- "soc_p50_pct":35,
- "soc_p10_pct":25,
- "soc_p90_pct":50},
- "..."
- ],
- "constraints": {
- "p_mic_import_kw": 80,
- "p_mic_export_kw": 35,
- "p_mic_margin_kw": {
- "base":5, "by_band_hour":[/*len=24*/]
- },
- "soc_hard_min_pct": 10,
- "soc_hard_max_pct": 95
- },
- "allow_sell": true,
- "allow_buy_grid": false,
- "ttl_sec": 1800,
- "advisory_weight": 0.0
- }
- schema_version
- 协议版本号，向后兼容
- soc_band P10/P50/P90
- 概率走廊（不是精确轨迹）
- allow_sell / allow_buy_grid
- 解锁套利项 A-3 / B-2
- p_mic_margin_kw
- 动态 margin：base + by_band_hour（定长 24）
- advisory_weight
- 0 = 影子模式；1 = 完全跟随
- 11 / 22

Notes:
- 11

## Slide 12
- 03a
- advisory_weight
- 灰度上线
- （
- TBD
- ）
- 影子模式 → 试套利 → 常规套利 → 完全跟随
- v0.3 · 技术交流
- weight = 0.0
- 0.0
- 影子模式
- 云发但边缘不动作，计算 delta 上行
- 1–2 周
- weight = 0.3
- 0.3
- 试套利
- A-3/B-2 小幅启用，密切监控 MIC
- 2–4 周
- weight = 0.7
- 0.7
- 常规套利
- 套利项常规启用，评估经济性
- 1 月+
- weight = 1.0
- 1.0
- 完全跟随
- advisory 完全生效，准备 V2G 扩展
- 稳态
- weight 怎么作用？
- 线性缩放套利项功率上限：K_max_A3 = 60 − P_bess_dis_baseline，套利上限 = weight · K_max。
- weight 不影响 A-1 / A-2 / B-1 等安全相关本地动作。
- weight=0 即影子：云照常发，本地按 A/B/C/D 跑，同时算『假如 weight=1 会动哪里』上行 delta。
- 12 / 22

Notes:
- 12

## Slide 13
- 02e
- 三档时间常数
- （
- TBD
- ）
- 30 min / 1 s / 200 ms / 事件 — 从慢到快的责任传递
- v0.3 · 技术交流
- 30 min
- 引导，非控制律
- L3 · 云端 advisory
- advisory 刷新（SOC band, allow_*, margin）
- 1 s
- 本地主决策
- L2 · 协调环
- A/B/C 决策 + D-1 + D-5 校验
- 200 ms
- 硬约束兜底
- L1 · D 快环
- D-2 MIC 守门 + D-3 BESS 物理 + D-4 BMS
- 事件
- 立即响应
- L1' · 中断
- BMS 一级告警 → SAFE_HOLD（<100 ms）
- 13 / 22

Notes:
- 13

## Slide 14
- 02f
- 控制环伪代码摘要
- L1 快环（200 ms）+ L2 协调环（1 s）— 两个独立 asyncio task
- v0.3 · 技术交流
- L1 快环 · 200 ms · D 守门
- # 每 200 ms
- 读 P_PoC (0x4022), P_bess (0x0716),
- SOC (0x075F), BMS 告警 (0x0764)
- # D-4 BMS 告警（事件 + 兜底）
- if BMS 一级告警:
- P_bess = 0
- → HARD_HOLD
- # D-2 MIC 守门（反馈式）
- margin = base
- if 高活动窗:
- margin = max(margin, band_hour[h])
- if P_PoC > MIC_import − margin:
- overshoot = P_PoC − (MIC − margin)
- 等比剪：P_gun_pool, P_bess_chg
- if -P_PoC > MIC_export − margin:
- overshoot = -P_PoC − (MIC_exp − margin)
- 等比剪：P_bess_dis, P_pv_setpoint
- # D-3 BESS 物理边界
- if SOC ≤ hard_min: P_bess_dis = 0
- if SOC ≥ hard_max: P_bess_chg = 0
- L2 协调环 · 1 s · A/B/C 决策
- # 每 1 s
- STEP 0  测量 + 读 advisory cache
- STEP 1  D 自检（BMS / 多桩 / PoC 链路）
- → 触发 → SAFE_HOLD
- STEP 2  C 列表：PV 利用
- P_to_EV_from_pv = min(P_pv, Σ gun_req)
- P_to_load = min(剩余 P_pv, P_load)
- P_pv_surplus = ...
- STEP 3  A 或 B 自动判定
- P_residual = (Σ gun + P_load) − PV 已供
- if P_residual > 0:
- P_bess_dis = min(60, P_residual)  # A-1+A-2
- P_grid_in = ...  # 不够下发 pool 给桩
- else:
- P_bess_chg = min(30, P_pv_surplus)  # B-1
- STEP 4  套利（soc_p10/p50/p90）
- K_max_A3 = 60 − P_bess_dis_baseline
- P_A3 = weight · K_max_A3  # 不影响 A-1/A-2/B-1
- STEP 5  D-1 负荷优先校验
- STEP 6  D-2 MIC 预校验（L1 会再剪）
- STEP 7  写 Modbus + 上行（含 shadow delta）
- 14 / 22

Notes:
- 14

## Slide 15
- 02f+
- 符号与单位约定
- 所有功率符号、单位、分辨率 — 避免 PoC / PCS / BESS / PV / 枪侧歧义
- v0.3 · 技术交流
- 电网侧 + 站级
- P_PoC
- 进网为 +、出网为 −
- kW
- 200ms
- 电表 0x4022 实测；MIC 守门以此为准
- P_pcs
- AC→DC 充 +、DC→AC 放 −
- kW
- 200ms
- 0x0716；PCS 既是 AC↔DC 接口也是 DC 母线电压源
- P_load
- 总为 +
- kW
- 1s
- 站内 AC 负荷（照明 / HVAC / 安防）
- P_gun_pool
- EMS 下发总盘子，+
- kW
- 1s
- OCPP 站控 per-枪分配；EMS 不感知 per-gun
- BESS / PV / 枪
- P_bess
- 充 +、放 −
- kW
- 1s
- BMS 上报；与 P_pcs 同号但 ≤ |P_pcs|（含损耗）
- SOC
- 0–100
- %
- 1s
- 0x075F；soc_p10/p50/p90 由云 advisory 给
- P_pv
- 发出为 +
- kW
- 1s
- 0x0706；DC/DC 直挂母线
- P_gun_i
- 单枪实际 +
- kW
- 1s
- 0x0900–09FF；EMS 只用聚合 Σ，不下发 per-gun
- 15 / 22

Notes:
- 15

## Slide 16
- 04
- 状态机：
- 6
- 系统态
- M0–M6 mode 真值表已取消；mode 改为日志侧的派生标签
- v0.3 · 技术交流
- 正常路径
- 异常 / 降级 / 兜底
- BOOT
- 上电、自举、加载配置
- SELF_TEST
- 组件握手 + Modbus 自检 + 云链路 + SOC ∈ [hard_min+5%, hard_max−5%]
- NORMAL
- 自检通过 + 云通讯正常；L2 协调环本地自主跑
- DEGRADED
- 单点故障；降额运行，剔除故障组件
- OFFLINE_AUTO
- 云断 ≥ 5 min；默认保守，TTL 后切默认表
- SOFT_HOLD
- MIC 偶发 / BMS 二级 / 桩丢包 → 30 s 自动重试
- HARD_HOLD
- BMS 一级 / 多桩故障 / DC 母线异常 → 人工复位
- 电表失联分级：
- 偶发 < 30s → 不降级 + 加大 margin　|　30s–2min → SOFT_HOLD 重连　|　> 2min → HARD_HOLD
- 16 / 22
- ISLAND_AC
- 电网外掉
- /
- PCS
- 失能 → 站内
- AC
- 断电，桩辅助受影响，
- EV
- 仅
- PV+BESS
- 经
- DC
- 母线（
- UK
- 概率低，兜底）

Notes:
- 16

## Slide 17
- 02g
- Modbus 寄存器映射
- 沿用上一代 + 新增 EV / MIC / 健康字 / advisory 通道
- v0.3 · 技术交流
- 沿用（来自原 telcoman）
- 0x4022
- 电表总功率（PoC）
- 0x0716
- PCS 当前功率
- 0x075F
- BESS SOC
- 0x0706
- PV 功率
- 0x0764
- BMS 一级告警
- 0x001B
- 方向 / 目标功率
- 新增（建议）
- 0x0900–09FF
- 各充电枪状态 / 功率 / P_max
- 0x0A00
- PV 限功率设定
- 0x0A10
- BESS 充/放设定（有符号）
- 0x0A20
- MIC 实时进/出 + 余量
- 0x0A30
- EMS 健康字 + 云链路状态
- 0x0A40
- SOC 跟踪误差 + advisory_weight
- 17 / 22

Notes:
- 17

## Slide 18
- 04a
- 可观测性（Observability）
- 7×24 营运的基础：出问题 5 分钟内能定位是云模型错还是本地动作错
- v0.3 · 技术交流
- {}
- 结构化日志
- 回放与定位
- 每协调环输出 JSON：SOC、各 setpoint、advisory_weight、TTL 剩余
- !
- 关键事件埋点
- 告警 + 复盘
- SOFT/HARD HOLD 进出、MIC 越限趋势、advisory 拒绝采纳、桩握手失败、SOC 偏出 P10/P90
- %
- 关键 KPI（云端聚合）
- 健康度评估
- 会话成功率、SOC 偏出 band 时长占比、MIC 触发次数、advisory 跟踪误差中位
- Δ
- Shadow delta
- 云模型验证
- weight=0 时上行『假如 weight=1 会动哪里』的差异
- 排查原则：
- 云模型错（SOC delta 大但本地动作合理）vs 本地动作错（本地动作偏离 A/B/C/D 规则）— 5 分钟内分清。
- 18 / 22

Notes:
- 18

## Slide 19
- 04b
- 讨论件（不在 EMS 范围，需专题立项）
- 通讯安全 / 接地与合规 / 接口语义 — 现状 + 待决策 + 负责人
- v0.3 · 技术交流
- 通讯安全
- MQTT TLS / 证书
- 现状：明文上行 ThingsBoard。待决策：双向 TLS + 证书轮转方案。
- 负责：云平台 + 安全
- Modbus 防伪指令
- 现状：寄存器读写无鉴权。待决策：是否加 ACL 白名单 / VLAN 隔离。
- 负责：运维 + 网络
- 远程升级回滚
- 现状：OTA 流程未定义。待决策：A/B 分区 + 灰度发布策略。
- 负责：EMS + DevOps
- 接地与合规 / 接口语义
- 接地与浪涌
- 现状：DC 母线接地方案未冻结。待决策：IT/TN-S 选择 + SPD 等级。
- 负责：电气 + 现场
- 英标合规 G99 / G100
- 现状：PCS 自带 G99 但未对站级出口测试。待决策：哪一项由站集成方报检。
- 负责：PCS 厂商 + EPC
- OCPP 接口语义
- 现状：P_gun_pool_max 单位 / 刷新率未对齐。待决策：与 SiteSupervisor 选型方共同 freeze。
- 负责：EMS + OCPP 选型方
- 19 / 22

Notes:
- 19

## Slide 20
- 04c
- 开放问题（v0.3 状态）
- v0.2 遗留问题在 v0.3 拍板情况；✓ 已闭环 / ? 待办
- v0.3 · 技术交流
- 已闭环 ✓
- ✓ DC 母线电压控制归属
- 划归 PCS 厂商；EMS 不参与
- ✓ advisory_weight 语义
- 线性缩放套利项；不影响 A-1/A-2/B-1
- ✓ by_band_hour 结构
- 定长 24 数组
- ✓ BESS 功率符号
- 充 +、放 −（与 P_pcs 同号）
- ✓ OCPP 总盘子边界
- EMS 只下发 P_gun_pool_max，per-枪由站控
- 待办 ?
- ? PV 容量定型
- 等场地确认；影响 B-1 / C-1 上限
- ? MIC base margin 默认值
- 5 kW 试运行 1 月后回调
- ? 孤岛模式触发判据
- 电网失压检测时间窗（<2s vs <5s）待 PCS 厂商确认
- ? Shadow delta 上报频次
- 每协调环 vs 每分钟聚合 — 看带宽
- ? MQTT / Modbus / 合规
- 见 04b 讨论件，task #8 跟踪
- 20 / 22

Notes:
- 20

## Slide 21
- 05
- 新旧关键差异
- 决策模型 / 状态机 / 接口 / 可扩展性
- v0.3 · 技术交流
- 维度
- 上一代 telcoman（光储能）
- 新一代 v0.3（光储充）
- 决策模型
- 6 布尔输入 → 7 mode 真值表（离散）
- A/B/C 优先级 + D 兜底（连续）
- 云端职责
- 下发 mode 索引 + 数值参数
- 下发 SOC band + allow_* + weight
- 本地自治
- 弱：mode 由云选择
- 强：A/B/C/D 本地跑，云只解锁套利
- 状态机
- M0–M6 共 7 mode 作控制指令
- 6 系统态（含 ISLAND_AC）+ SOFT/HARD HOLD 双档
- MIC 守门
- 上下限阈值
- 200 ms 反馈式，以 PoC 为准，不预测
- EV 充电
- 无
- OCPP 协议 + 总盘子下发，桩自分配
- 影子模式
- 无
- advisory_weight=0 即影子，记录 delta
- 可扩展性
- 加场景需扩真值表
- 加场景只在 A/B/C 加项
- 22 / 22
- 孤岛模式
- 无（隐含
- OFFLINE_AUTO）
- 新增
- ISLAND_AC：
- 电网外掉
- /
- PCS
- 失能时站内
- AC
- 断电兜底
- DC
- 母线电压控制
- EMS
- 与
- PCS
- 边界含糊
- 明文划归
- PCS
- 厂商；
- EMS
- 不参与电压环

Notes:
- 21
