# Session Source Ledger

Last updated: 2026-05-15

用途：保存本轮 EV 场站需求分析 session 里出现过、用过、讨论过的主要来源。这里比“文献库”更大，不只包含论文/报告，也包含用户截图、监控数据、脚本、PPT、公开数据、商业 API 讨论和内部输出。

## 使用原则

- `Primary / official`：可以作为较强引用来源。
- `Local observed data`：可以作为本项目实际证据，但要说明采集口径和限制。
- `Proxy / benchmark`：可以支撑方向或假设，不能当精确实测。
- `User-provided`：可以作为业务背景或人工验证材料，需要和外部/本地数据互相校验。
- `AI-assisted`：Gemini / ChatGPT / Poe 只能作为检索和整理工具，不能作为文献来源。

## 当前最重要的来源分组

| Group | 用途 | 主来源 |
| --- | --- | --- |
| Charger category | 解释快充/慢充为什么分层 | GOV.UK, EVA England, Zapmap legend/user screenshots |
| Power delivery ratio | 解释额定功率为什么要折减 | ICCT, P3, GIREVE, DOE appendix |
| Local utilisation proxy | 证明附近真实充电行为 | Zapmap monitor outputs, SQLite/CSV, Station Yard/Marnwood reports |
| Site capture logic | 解释新站为什么只能 capture 一部分区域需求 | DfT AADF, POI/site context, competition, site friction, product fit |
| Grid/site feasibility | 解释需求之外还要看接入和施工 | UKPN, IET, DNO open data, site visit/PPT |
| Macro calibration background | 解释旧模型和区域模型边界 | Dundee/ChargePlace Scotland, DfT vehicle stock, census outputs |

## Source Table

| ID | Source | Type | Saved / URL | Used For | Status | Do Not Use For |
| --- | --- | --- | --- | --- | --- | --- |
| S001 | User screenshots: Ironbridge / Station Yard / Zapmap app status/key | User-provided | Chat attachments; summarized in project files | Manual location verification, Zapmap legend, Station Yard/Wharfage/KFC context | Background / manual evidence | Do not use as raw machine-readable dataset |
| S002 | User PPT: `ironbridge - site visit 20260511(1).pptx` | User-provided | Original path in chat; assets extracted under `tmp/ironbridge_ppt_assets/` | Site context, site visit background, candidate-site narrative | Background | Do not use as final measured demand |
| S003 | Zapmap web-map / status probe | Local observed data / web POC | `docs/2026-05-10-zapmap-webmap-feasibility-note.md`; `docs/2026-05-10-zapmap-webmap-probe-implementation.md`; `scripts/zapmap_status_probe.py`; cloud monitor repo | 15-minute status monitoring; Available / Charging / Faulted / Unknown proxy | Core local utilisation proxy | Do not call it licensed production API or CPO meter kWh |
| S004 | Zapmap probe sample CSV | Local observed data | `data/zapmap_probe/zapmap_probe_evse_20260509T161500Z.csv`; `data/zapmap_probe/zapmap_probe_summary_20260509T161500Z.csv` | First web-map feasibility snapshot | Historical POC | Do not use as current 24h demand |
| S005 | EV utilisation monitor outputs for Ironbridge/Marnwood | Local observed data | `output/share/ironbridge_ev_stage0/`; `output/share/ironbridge_ev_stage0/data_refresh_2026-05-14/*.csv`; `marnwood_throughput_bottom_table.*` | Current local demand proxy, heatmap, station contribution, throughput table | Active working source | Re-run before final external report |
| S006 | VPS / SQLite monitor database | Local observed data | Cloud-side `ev-utilisation-monitor`; local docs mention `data/ev_monitor.sqlite` | Scheduled 15-minute snapshots and normalized data model | Active operational source | Do not overclaim if latest row count not refreshed |
| S007 | Eco-Movement OCPI Data API user guide | Commercial API reference | User pasted excerpt in chat; source should be re-linked before external use | Explains commercial OCPI/PATCH/GET route and why web scraping is POC only | Commercial alternative reference | Do not imply we have a paid subscription unless confirmed |
| S008 | OCPI protocol concept | Protocol reference | Mentioned through Eco-Movement guide | Explains standard EV roaming/location/status data model | Background | Do not use as actual local data source without API access |
| S009 | GOV.UK EV charging infrastructure statistics FAQ | Official reference | https://www.gov.uk/government/publications/electric-vehicle-charging-device-statistics-information/electric-vehicle-charging-infrastructure-statistics-frequently-asked-questions | Charger power-band categories, rapid/ultra-rapid distinction | Strong for classification | Not local demand evidence |
| S010 | EVA England charger type guide | Industry/user education | https://www.evaengland.org.uk/about-electric-vehicles/about-charging/types-of-ev-chargepoints/ | Plain-English charger type explanation | Useful for business explanation | Not numerical throughput evidence |
| S011 | ICCT 2024 EV charger infrastructure report | Independent research report | `docs/references/source_pdfs/ICCT_2024_US_EV_chargers_2032_power_delivery_ratio.pdf`; original URL in literature registry | Main source for power delivery ratio assumptions | Strong assumption support | Not UK/Marnwood measured data |
| S012 | P3 Charging Index 2024 | Charging benchmark report | `docs/references/source_pdfs/P3_Charging_Index_2024.pdf`; original URL in literature registry | Explains peak power vs average charging power | Strong explanation support | Not site throughput model |
| S013 | GIREVE Beyond EV Charging #15 | Market/session insight | https://www.gireve.com/beyond-ev-charging-15/ | Supports fast-charger preference and average delivered power below max | Supporting evidence | Not exact Marnwood delivery ratio |
| S014 | DOE Fact of the Week #1319 | Public agency data summary | https://www.energy.gov/node/4835693 | Appendix sanity check for broad DC session averages | Appendix only | Not main parameter source |
| S015 | DfT AADF 2024 traffic data | Official/local traffic data | `/Users/wuchenghan/Documents/New project 2/data/public_ev_site_samples/processed/dft_aadf_2024.csv`; raw CSVs under `data/public_ev_site_samples/raw/` | Traffic access layer for capture-rate logic | Strong for road-link traffic evidence | Does not prove vehicles stop/charge |
| S016 | DfT vehicle stock / VEH outputs | Official/local vehicle data | `微网数电/充电桩需求估计/07_验证结果/dft_vehicle_stock.csv` | Macro EV/car-stock background | Useful background | Not single-site utilisation |
| S017 | ChargePlace Scotland / Dundee session data | Local validation dataset | `微网数电/充电桩需求估计/03_数据/external/chargeplace_scotland/`; `07_验证结果/dundee_*` | Old regional calibration and Dundee holdout evidence | Strong for macro calibration boundary | Not direct Marnwood demand |
| S018 | Scotland / Dundee census outputs | Public demographic data | `微网数电/充电桩需求估计/03_数据/external/scotland_dundee_census_2022.csv`; `dundee_scotland_census_source.csv` | Macro demand factor and household context | Background/calibration | Not current site-level capture |
| S019 | Regional calibration verification outputs | Local model evidence | `微网数电/充电桩需求估计/07_验证结果/verification_report.md`; `macro_model_backtest.csv`; `session_serviceability_results.csv`; `acceptance_checklist.md` | Old model validation and comparison baseline | Background model evidence | Do not present as Marnwood result |
| S020 | DNO / grid open-data samples | Local data | `data/dno_grid_capacity/processed/*.csv`; `tmp/openags-k-bess-dno/literature/notes/k-bess-dno-integration.md` | Grid headroom / feasibility background | Early evidence | Not a formal connection quote |
| S021 | UKPN Charge Collective Final Report | Grid / network report | `docs/references/source_pdfs/UKPN_Charge_Collective_Final_Report.pdf` | Network feasibility and grid constraint framing | Good background | Does not set capture rate |
| S022 | GOV.UK connecting EV chargepoints to electricity network | Official guidance | https://www.gov.uk/government/publications/connecting-electric-vehicle-chargepoints-to-the-electricity-network/connecting-electric-vehicle-chargepoints-to-the-electricity-network | DNO/installer process and grid-connection caveat | Strong guidance | Not local headroom |
| S023 | UKPN business EV hub | Network operator guidance | https://www.ukpowernetworks.co.uk/low-carbon-technology-commercial/business-electric-vehicle-hub | UKPN connection/process context | Strong process source | Not local quote |
| S024 | IET Guide to EV Charging Infrastructure for Local Authorities | Technical guide | `docs/references/source_pdfs/IET_Guide_to_EV_Charging_Infrastructure_for_Local_Authorities.pdf` | Infrastructure planning and local authority context | Useful planning support | Not local kWh estimate |
| S025 | Pod Point / Savills case study | Case material | `docs/references/source_pdfs/Pod_Point_Savills_Case_Study.pdf` | Property/site deployment case framing | Case reference | Not numerical benchmark for Marnwood |
| S026 | Hometrack Site Appraisal sample | Report structure sample | `docs/references/source_pdfs/Hometrack_Site_Appraisal_Report_Sample.pdf` | How to structure site appraisal evidence | Format reference | Not EV-specific demand source |
| S027 | Rivervale website | Site source | https://www.rivervale.co.uk/contact-us | Rivervale site identity and business context | Used in Rivervale project | Not Marnwood evidence |
| S028 | GetTheData BN41 1XB | Location/postcode source | https://www.getthedata.com/postcode/BN41-1XB | Rivervale postcode coordinate support | Used in Rivervale project | Not precise surveyed coordinate |
| S029 | Electric Brighton Victoria Road rapid hub | Local charger source | https://electricbrighton.com/victoria-road-rapid-hub | Rivervale nearby competition context | Used in Rivervale project | May be stale; verify current CPO/Zapmap |
| S030 | Zapmap location pages | Web charger pages | Example in Rivervale ledger; Zapmap station pages | Charger inventory, devices/connectors, pricing where visible | Supporting public web source | Status/pricing can change |
| S031 | Google / Apple Maps / Tencent screenshots | Map/user context | User screenshots and map views | Manual geography, POI, distance intuition | Background | Not formal geospatial dataset |
| S032 | Google Places API outputs | Leadgen/site discovery data | `data/google_places_*`; `微网数电/data/london_car_wash_*`; scripts under `leadgen/` | Earlier car-wash leadgen and POI/site discovery pattern | Useful for POI/catchment enrichment | Not charger utilisation evidence |
| S033 | Companies House API outputs | Company enrichment | `data/london_car_wash_companies_v1.csv`; `微网数电/data/london_car_wash_companies_v1.csv` | Leadgen company matching | Leadgen only | Not EV demand source |
| S034 | Stage-0 PPT reference pack | Local reference summary | `docs/references/2026-05-14-stage0-ppt-reference-pack.md` | Links report/PPT reference material | Internal reference | Verify exact source before external quoting |
| S035 | AI tools: Gemini / ChatGPT / Poe | AI-assisted search/synthesis | Chat-only; no source authority | Helped brainstorm, search, summarize | Tool only | Never cite as literature/source |

## 如何对应到报告里的三类问题

### 1. 快充和慢充分层

Use:

- S009 GOV.UK
- S010 EVA England
- S001 Zapmap app legend screenshot as manual/context support

Can say:

```text
我们按功率和使用场景把充电桩分层。候选方案是 160kW 双枪快充，因此主分析优先看 50kW 以上 rapid / ultra-rapid 层，而不是把 7kW / 22kW AC 慢充混入平均。
```

Cannot say:

```text
GOV.UK / EVA 证明这个站点有需求。
```

### 2. 额定功率和实际功率

Use:

- S011 ICCT as main assumption source
- S012 P3 as peak-vs-average explanation
- S013 GIREVE as market/session support
- S014 DOE as appendix only

Can say:

```text
125kW / 160kW 是额定能力，不能直接等同于全程平均输出。参考 ICCT 的 power delivery ratio，125-160kW 级别主测算可用 60%-65%，保守情景用 50%。
```

Cannot say:

```text
DOE 证明 Marnwood 应该用 35%-60%。
```

### 3. Capture rate

Use:

- S005/S006 for observed regional demand proxy
- S015 for traffic access
- S031/S032 for POI and dwell reason
- S003/S030 for competition and charger inventory/status
- S020-S024 for grid/site feasibility boundary

Can say:

```text
Capture rate 不是固定文献常数，而是新站竞争能力判断。我们按交通可达、停留理由、竞品、车位/入口摩擦、产品配置、用户类型拆成 conservative / base / upside 三档。
```

Cannot say:

```text
某篇文献直接给了 Marnwood 的 capture rate。
```

## 当前仍缺的来源

| Missing Source | Why Needed | Owner / Route |
| --- | --- | --- |
| Marnwood site access / bay count / ownership | Capture rate and buildability | BD / site owner |
| Marnwood exact DNO quick scan or connection quote | Feasibility and capex | Installer / DNO |
| Weekend 7-day utilisation window | Annualisation and weekday/weekend adjustment | Monitor |
| Formal commercial API / licensed data route | Production/compliance | Eco-Movement / Zapmap commercial / OCPI route |
| Pricing and payment assumptions for new station | Revenue model | Finance / product |

