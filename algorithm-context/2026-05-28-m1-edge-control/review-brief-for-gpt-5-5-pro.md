# Review Brief For GPT-5.5 Pro

## Task

Please review the M1 local-control algorithm context and propose a cleaner v0.2 formula.

Important: read `source-notes/ning-edge-control-strategy-draft-02.txt` and `source-notes/ning-edge-control-strategy-draft-02-update-note.md` before relying on draft 01. Ning indicated the earlier draft was wrong, and draft 02 changes the core price formula.

Latest local draft to review:

```text
m1-local-algorithm-brief-v0.2.md
```

This draft has already incorporated two review passes on physical caps and algorithm interface. Please focus on remaining correctness and business-policy questions, not on v0.1.

The formula must stay narrow and implementable:

```text
BESS + EV
Import Only
No PV
No export
No V2G
```

Do not expand the scope into full EMS optimization.

## Current Fixed Output

Keep this output contract unless there is a strong reason to change it:

```python
Y_t = {
    "mode": "NORMAL / BESS_SUPPORT / EV_LIMIT / BESS_CHARGE / SAFE_PROTECT",
    "p_ev_limit_kw": 0.0,
    "p_bess_target_kw": 0.0,
    "reason_code": "NORMAL / EV_DEMAND_HIGH / MIC_LIMIT / SOC_LOW / PCS_LIMIT / BESS_FAULT / DATA_STALE",
}
```

Notes:

- `p_ev_limit_kw` maps to site-level `P_gun_pool_max`.
- `p_bess_target_kw > 0` means charge.
- `p_bess_target_kw < 0` means discharge.
- `p_bess_target_kw = 0` means idle.

## Historical No-Price v0.1 Formula

```text
P_grid_available = max(0, MIC - MIC_margin - site_load)

EV_gap = max(0, EV_request - P_grid_available)

BESS_available = min(
  PCS_discharge_limit,
  BMS_discharge_limit,
  SOC_discharge_limit
)

if BESS protection not OK:
  BESS_available = 0

if SOC < soc_p25:
  BESS_available = 0

P_bess_support = min(EV_gap, BESS_available)

p_ev_limit_kw = min(EV_request, P_grid_available + P_bess_support)

p_bess_target_kw = -P_bess_support
```

Optional charge branch:

```text
if EV_gap = 0 and SOC <= soc_p25 and charge is allowed:
  p_bess_target_kw = +P_bess_charge
```

## Required v0.2 Improvement

Add economic dispatch without making the edge algorithm complex.

Use the draft 02 pattern:

```text
physical_cap -> utilization_ratio -> target_power
```

Avoid additive price bonuses that can exceed physical caps:

```text
physical_cap + price_bonus_kw
```

Cloud should provide:

```text
soc_p10_t
soc_p25_t
soc_p50_t
soc_p75_t
soc_p90_t
grid_buy_price_rank_t
ev_charge_price_rank_t
spread_rank_t
```

Interpretation:

```text
grid_buy_price_rank_t = 0 cheap grid power, 1 expensive grid power
ev_charge_price_rank_t = 0 low EV charging revenue, 1 high EV charging revenue
spread_rank_t = 0 poor spread, 1 attractive spread
```

M1 does not use:

```text
export_sell_price
```

because there is no export and no V2G.

## Boundaries To Preserve

Do not make these our responsibility:

| Layer | Boundary |
|---|---|
| L1 protection | Existing MIC / BMS / PCS / SOC / data-freshness protection provides status and limits |
| Gun allocation | IT / charger side allocates `P_gun_pool_max` across guns/modules |
| Raw price engine | Cloud computes ranks; edge consumes ranks |
| Export/V2G | Out of scope |

## What Needs Review

Please propose:

1. A v0.2 mathematical formula that combines:
   - MIC headroom
   - EV gap
   - SOC band
   - price rank / spread rank
   - BESS limits

2. A simple rule for when to:
   - support EV by discharging BESS
   - charge BESS
   - limit EV
   - hold
   - protect

3. A minimal input table for IT:
   - field name
   - source
   - granularity
   - required vs optional

4. A test matrix for integration:
   - low SOC + cheap price
   - low SOC + expensive price
   - high SOC + EV gap
   - normal SOC + EV gap + high spread
   - MIC tight
   - BESS unavailable
   - data stale

## Avoid

- Do not introduce a full optimizer on the edge.
- Do not solve per-gun allocation in this formula.
- Do not use export price in M1.
- Do not require raw price tables on the edge.
- Do not change the output contract unless necessary.
