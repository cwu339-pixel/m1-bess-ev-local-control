# EV Charging Literature Registry

Last updated: 2026-05-15

用途：把目前 EV 场站需求分析里用到的文献和参考来源统一保存，避免结论只停留在聊天里。

整个 session 的完整来源总账见：

```text
docs/references/session-source-ledger-2026-05-15.md
```

## 怎么用这份文献库

这份表不是为了堆材料，而是为了回答三个业务问题：

1. 为什么快充和慢充要分开？
2. 为什么不能直接用额定功率计算 kWh？
3. 哪些资料只能作为旁证，不能直接变成模型参数？

注意：Gemini / ChatGPT / Poe 这类 AI 工具只能作为检索、整理或草稿辅助，不能作为报告里的文献来源。对外引用时应引用 GOV.UK、ICCT、P3、GIREVE、DOE 等原始来源。

## 核心结论和对应来源

| 结论 | 主来源 | 支撑强度 | 在报告里怎么用 |
| --- | --- | --- | --- |
| 快充和慢充应分开，不应混合平均 | GOV.UK EV charging infrastructure statistics FAQ；EVA England charger type guide | High | 支撑 `>=50kW` fast-charging benchmark layer |
| 125kW / 160kW 是额定功率，不等于全程实际输出 | ICCT 2024 charger report；P3 Charging Index；GIREVE #15 | High | 支撑 power delivery ratio |
| 125-160kW 快充可用 50%-65% 做敏感性边界，主表可只放 50%-60% | ICCT 2024 charger report + 本项目工作假设 | High for boundary / Medium for midpoint | `65%` 来自 ICCT 150kW；`50%` 来自 ICCT 高功率保守边界；`60%` 是中间工作假设，不是文献原文 |
| peak/max charging power 通常只维持一小段时间 | P3 Charging Index | Medium-high | 用来解释为什么要看 average charging power |
| 真实 DC session 的平均输出可能显著低于最大功率 | GIREVE #15；DOE FOTW #1319 | Medium | 作为旁证，不直接当 Marnwood 主参数 |
| site selection / DNO / local authority planning 需要单独证据 | IET guide；UKPN report；Pod Point/Savills case material | Medium | 用于后续 site feasibility，不用于直接算 kWh |

## 文献明细

### L001 - GOV.UK EV charging infrastructure statistics FAQ

- Source type: official government reference
- URL: https://www.gov.uk/government/publications/electric-vehicle-charging-device-statistics-information/electric-vehicle-charging-infrastructure-statistics-frequently-asked-questions
- Local copy: not downloaded; web source
- Supports:
  - UK public charging statistics use power-band descriptors such as Standard, Standard plus, Rapid, Ultra-rapid.
  - This supports separating slow/standard, standard-plus, rapid, and ultra-rapid charger layers.
- How to use:
  - Use for charger power-band definitions.
  - Use to justify that `>=50kW` should be treated as rapid/ultra-rapid rather than mixed with low-power AC.
- Caveat:
  - This supports classification, not demand or revenue.

### L002 - EVA England charger type guide

- Source type: UK EV user/industry education source
- URL: https://www.evaengland.org.uk/about-electric-vehicles/about-charging/types-of-ev-chargepoints/
- Local copy: not downloaded; web source
- Supports:
  - Slow, fast, rapid, and ultra-rapid chargers have different power ranges and use cases.
  - Rapid chargers are typically 50kW to 149kW; ultra-rapid is 150kW and above.
- How to use:
  - Use as plain-English support for explaining fast vs slow charger behavior to business people.
- Caveat:
  - Not a numerical throughput study.

### L003 - ICCT 2024 U.S. EV charger infrastructure report

- Source type: independent research report
- Organization: ICCT, International Council on Clean Transportation
- URL: https://theicct.org/wp-content/uploads/2024/03/ID-89-%E2%80%93-Chargers-2032-Report-letter-70112-v6-1.pdf
- Local copy: `docs/references/source_pdfs/ICCT_2024_US_EV_chargers_2032_power_delivery_ratio.pdf`
- Supports:
  - DC fast charger `power delivery ratio` should be lower than 100%.
  - Report assumption examples:
    - 50kW charger: 75% in 2023.
    - 150kW charger: 65% in 2023.
    - 350kW charger: 50% in 2023.
- How to use:
  - This is the main parameter source for Marnwood power delivery ratio.
  - For 125-160kW charger discussion, treat `65%` as the closest ICCT public reference point and `50%` as the conservative lower boundary.
  - Treat `60%` as this project's midpoint working assumption, not as a directly quoted ICCT number.
- Caveat:
  - It is a U.S. infrastructure modelling report, not a direct UK Marnwood measurement.
  - Use as assumption support, not as observed local performance.

### L004 - P3 Charging Index 2024

- Source type: independent engineering/automotive charging benchmark
- URL: https://www.p3-group.com/en/p3-charging-index/
- PDF URL: https://www.p3-group.com/wp-content/uploads/2024/12/P3_Charging-Index_EN.pdf
- Local copy: `docs/references/source_pdfs/P3_Charging_Index_2024.pdf`
- Supports:
  - Maximum charging power is not the same as average charging power.
  - P3 compares average charging power because official maximum performance is typically only reached for a few minutes in a session.
- How to use:
  - Use as explanation source: why `rated/peak power × duration` is too optimistic.
  - Good for non-technical readers.
- Caveat:
  - It is vehicle charging-performance benchmarking, not a site revenue or CPO throughput model.

### L005 - GIREVE Beyond EV Charging #15

- Source type: CPO/eMSP roaming and transaction-data insight
- URL: https://www.gireve.com/beyond-ev-charging-15/
- Local copy: not downloaded; web source
- Supports:
  - In locations with multiple DC power levels, drivers often prefer faster options.
  - Despite advertised maximum-power differences, average delivered power remains much lower than maximum capacity.
- How to use:
  - Use to support two ideas:
    - fast chargers should be benchmarked separately;
    - high maximum power still needs delivery-ratio adjustment.
- Caveat:
  - Useful as market/session evidence, but not enough to define Marnwood's exact delivery ratio by itself.

### L006 - DOE Fact of the Week #1319

- Source type: U.S. public agency data summary
- URL: https://www.energy.gov/node/4835693
- Local copy: not downloaded; web source
- Supports:
  - Average paid DC fast charging session in the cited sample: 42 minutes and 22 kWh.
  - Implied broad session average power: about 31kW.
- How to use:
  - Appendix / sanity-check only.
  - Use to explain that full-session average power can be far lower than charger nameplate.
- Caveat:
  - Do not use as main Marnwood parameter.
  - It does not isolate 50kW / 150kW / 350kW charger classes for our site.

### L007 - IET Guide to EV Charging Infrastructure for Local Authorities

- Source type: technical / local authority infrastructure guide
- Local copy: `docs/references/source_pdfs/IET_Guide_to_EV_Charging_Infrastructure_for_Local_Authorities.pdf`
- Supports:
  - EV charging infrastructure planning needs site, network, user, and implementation considerations.
- How to use:
  - Use for site-feasibility and planning language.
  - Do not use for local kWh/day parameter.
- Caveat:
  - It is planning guidance, not local Marnwood demand evidence.

### L008 - UKPN Charge Collective Final Report

- Source type: electricity network / charging infrastructure project report
- Local copy: `docs/references/source_pdfs/UKPN_Charge_Collective_Final_Report.pdf`
- Supports:
  - Network and connection constraints need separate assessment from demand.
- How to use:
  - Use for DNO/grid feasibility discussion.
  - Keep separate from demand-pool calculation.
- Caveat:
  - Does not set capture rate.

### L009 - Pod Point / Savills Case Study

- Source type: charging deployment / property case material
- Local copy: `docs/references/source_pdfs/Pod_Point_Savills_Case_Study.pdf`
- Supports:
  - Property/site context matters for EV charging deployment.
- How to use:
  - Use as case-study style reference for site context and stakeholder explanation.
- Caveat:
  - Not a numerical demand forecast benchmark for Marnwood.

### L010 - Hometrack Site Appraisal Report Sample

- Source type: site appraisal sample
- Local copy: `docs/references/source_pdfs/Hometrack_Site_Appraisal_Report_Sample.pdf`
- Supports:
  - Good site appraisals separate location facts, evidence, assumptions, and conclusions.
- How to use:
  - Use as report-structure inspiration.
- Caveat:
  - Not EV-specific demand evidence.

## 当前 Marnwood 推荐引用口径

### 快充 / 慢充分层

Use:

- L001 GOV.UK
- L002 EVA England

Business wording:

```text
我们把快充和慢充分开，是因为它们对应不同的使用场景。慢充更像停车顺便充，快充更像专门补能。候选站点是 160kW 双枪快充，所以主分析应看 50kW 以上的 rapid / ultra-rapid benchmark，而不是把 7kW / 22kW AC 慢充混进去平均。
```

### 额定功率 vs 实际功率

Use:

- L003 ICCT as main source
- L004 P3 as explanation
- L005 GIREVE as market/session support
- L006 DOE only as appendix

Business wording:

```text
125kW / 160kW 是设备额定能力，不代表整段充电都按这个功率输出。实际输出会受车辆接收能力、SOC、温度和充电曲线影响。参考 ICCT 的 charger power delivery ratio，150kW 公开假设可到 65%，高功率保守边界可用 50%。本项目把 60% 作为中间工作假设；P3 和 GIREVE 也说明了 peak power 与 average delivered power 必须分开。
```

### Capture rate

Current status:

```text
Capture rate 还没有一个可以直接套用的统一标准文献。
```

当前做法应该是把 capture rate 作为商业情景参数，而不是伪装成文献定值。它应由以下因素共同解释：

| Layer | 要看什么 | 为什么影响 capture rate | 证据类型 |
| --- | --- | --- | --- |
| Traffic access | AADT、主路距离、进出是否顺路、能不能看见 | 车流不等于会停，但没有车流很难 capture | DfT AADF、地图、site photos |
| Dwell reason | 有没有餐饮、厕所、商店、酒店、景点、办公/车队停留理由 | 快充也要等 15-40 分钟，没有等待理由会降低转化 | POI、现场照片、Google/Apple Maps |
| Competition | 周边 2.5/5km 内快充数量、功率、价格、可靠性、是否繁忙 | 新站只能 capture 一部分区域需求，不是拿走全部需求 | Zapmap monitor、站点功率、价格 |
| Site friction | 车位数量、bay 类型、24h 可达性、是否容易倒车/停车 | 车能不能方便停进去，直接影响使用率 | site plan、现场照片、owner input |
| Product fit | 新站功率、双枪/单枪、支付便利、价格、是否有 BESS 支撑 | 产品比竞品弱，capture rate 要下调；产品更强才可上调 | 设备方案、价格假设、竞品比较 |
| User mix | 本地居民、游客、过路车、fleet/customer use | 不同用户停车目的和充电频率不同 | catchment、POI、BD 信息 |

建议先用三档 capture-rate 情景，不要装成一个“行业标准常数”：

| 情景 | Capture rate 方向 | 适用条件 | 解释方式 |
| --- | ---: | --- | --- |
| Conservative | 低 | 站点缺少强停留理由，竞品更方便，入口/车位一般 | 只能吃到少量外溢需求 |
| Base | 中 | 有一定交通和停车理由，产品功率接近竞品，但不明显优于 KFC/Shell | 能 capture 一部分区域快充需求 |
| Upside | 高 | 站点有强引流/强停车理由、接入便利、价格/功率/体验优于附近竞品 | 才能接近强 benchmark 的表现 |

Business wording:

```text
文献和数据可以支撑区域需求池和实际功率折减，但新站能拿到多少，必须通过 capture rate 做情景分析。Capture rate 不是一个固定行业常数，而是由站点位置、停留理由、竞争站、价格、可达性和用户类型共同决定。
```

更直白的汇报口径：

```text
快充/慢充分层和实际功率折减有外部文献可以支撑；capture rate 本质上是新站竞争能力判断。我们会把它拆成交通、停留理由、竞品、车位可达性、产品配置和用户类型几层，再做 conservative / base / upside 三档，而不是只拍一个数字。
```

## 不建议主线引用的方式

- 不要说 DOE 直接证明 Marnwood 应该用 35%-60%。
- 不要说 P3 给出了站点 throughput。
- 不要说 GIREVE 能直接给 Marnwood capture rate。
- 不要把 GOV.UK / EVA 的 charger category 当成需求证据。
- 不要把 ICCT 的 U.S. deployment assumption 说成 Marnwood 实测。
- 不要把 Gemini / ChatGPT / Poe 写成文献来源；AI 只能辅助整理，引用必须回到原始来源。
