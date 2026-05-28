# Review Brief For GPT-5.5 Pro

Date: 2026-05-28

## Task

Please review the regenerated M1 Model 1 v0.3 local-control algorithm.

Read these files first:

```text
m1-model1-explainer-v0.3.md
m1-model1-local-algorithm-v0.3.md
m1-model1-v0.3-expert-review.md
```

Then use these as background only:

```text
meeting-summary-2026-05-28.md
m1-four-control-loops-v0.1.md
source-notes/ning-edge-control-strategy-draft-02-update-note.md
source-notes/ning-edge-control-strategy-draft-02.txt
m1-local-algorithm-brief-v0.2.md
```

## Current Scope

Do not expand the problem.

```text
Model 1 = BESS + EV
Import Only
No PV
No export
No V2G
```

The current v0.3 framing is:

```text
L3 cloud parameters + L1 safety limits + L2 local realtime state
        -> local charge-side scheduling
        -> Y_t
```

## Fixed Output Contract

Keep this output unless there is a strong reason to change it:

```python
Y_t = {
    "mode": "NORMAL / EV_LIMIT / BESS_CHARGE / SAFE_PROTECT",
    "p_ev_limit_kw": 0.0,
    "p_bess_target_kw": 0.0,
    "reason_code": "NORMAL / MIC_LIMIT / SOC_LOW / PCS_LIMIT / BESS_FAULT / DATA_STALE",
}
```

Interpretation:

```text
p_ev_limit_kw maps to site-level P_gun_pool_max.
p_bess_target_kw > 0 means charge.
p_bess_target_kw < 0 is not emitted in Model 1.
p_bess_target_kw = 0 means idle.
```

## Important Current Decisions

1. **L1 / L2 / L3 are layers, not a serial pipeline.**

```text
L3 sends model parameters, SOC bands, and price ranks.
L1 sends safety status and hard power limits.
L2 computes targets inside those boundaries.
L1 should also clamp / reject final outputs before execution.
```

2. **Price logic uses the draft 02 structure.**

Use:

```text
physical_cap -> utilization_ratio -> target_power
```

Do not use:

```text
physical_cap + price_bonus_kw
```

3. **v0.3 is charge-side scheduling only.**

Model 1 does not implement BESS discharge:

```text
p_bess_target_kw >= 0
No discharge mode is active in Model 1
```

Do not add a discharge placeholder back into Model 1.

4. **Site-load口径 is a critical risk.**

The current v0.3 uses:

```text
site_base_load_kw
```

This must not include the current EV request or current BESS target. If the measurement口径 is unclear, prefer asking IT for:

```text
site_import_headroom_kw
```

## What To Review

Please provide:

1. Whether v0.3 is logically closed enough for IT/hardware integration discussion.
2. Whether the formula risks double-counting EV/BESS load.
3. Whether `site_import_headroom_kw` should become the preferred required input instead of computed MIC minus load.
4. Whether the output contract is enough for IT to test.
5. Whether the explanation version is clear enough for a non-control-engineering stakeholder to defend in a meeting.
6. Top 5 changes before sending this to Ning/IT.

## Avoid

- Do not propose a full optimizer on the edge.
- Do not solve per-gun allocation.
- Do not introduce export price or V2G.
- Do not require raw tariff tables on the edge.
- Do not require IT to implement event-driven discharge in Model 1.
- Do not change the output contract unless necessary.
