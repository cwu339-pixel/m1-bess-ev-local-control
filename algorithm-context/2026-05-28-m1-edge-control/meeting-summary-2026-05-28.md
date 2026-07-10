# 2026-05-28 Meeting Summary

## Short Conclusion

Ning did not reject the current M1 direction. He rejected treating the algorithm as a loose MVP.

The right framing is:

```text
The scenario should be narrow, but the formula should be correct.
```

## What Was Accepted

The current problem definition is mostly right:

```text
M1 = BESS + EV
Import Only
No PV
No export
No V2G
```

The output can stay simple:

```python
Y_t = {
    "mode": "...",
    "p_ev_limit_kw": 0.0,
    "p_bess_target_kw": 0.0,
    "reason_code": "...",
}
```

`p_ev_limit_kw` is the site-level EV pool limit and can map to `P_gun_pool_max`.

## Main Feedback From Ning

1. The current formula is missing price.

The no-price formula can run, but it is closer to:

```text
MIC protection + EV support
```

It is not yet a proper economic-dispatch formula.

2. L1 protection and gun allocation should not be mixed into our algorithm.

They should be documented as separate loops:

| Loop | Meaning | Our role |
|---|---|---|
| Protection loop | MIC / BMS / PCS / SOC / data freshness | Consume status and limits |
| Power aggregation loop | EV + AC aggregate load vs MIC | Use its measurements and constraints |
| Economic-dispatch loop | SOC band + price + EV/AC demand | This is our layer |
| Gun/module allocation loop | Split EV pool power across guns/modules | Review later, do not change in this version |

3. Price ranking should be included.

Cloud should derive price-ranking signals and push them down. Edge should not calculate raw prices.

Possible price layers:

| Price layer | Meaning | M1 status |
|---|---|---|
| Grid buy price | Cost of buying power from grid | In scope |
| EV charge price | Revenue / tariff charged to EV users | In scope |
| Export sell price | Price for selling power back to grid | Out of scope for M1 |

4. IT is waiting for the formula.

The IT side likely does not need large changes. Once the formula and required fields are fixed, they can provide real fields or simulated fields and start integration testing.

## Current Interpretation

Our layer should do this:

```text
cloud SOC bands + cloud price ranks + local MIC/headroom + local EV/AC load + L1 limits
        -> p_ev_limit_kw
        -> p_bess_target_kw
```

Our layer should not do this:

```text
raw tariff parsing
PCS/BMS hard protection
per-gun allocation
export optimization
V2G
```

## Next Work

Create v0.2 of the formula:

1. Keep the current MIC / EV gap / BESS available structure.
2. Add SOC band state using `soc_p10/p25/p50/p75/p90`.
3. Add price-ranking inputs.
4. Define charge and discharge scores.
5. Preserve the simple output contract.
6. Update the notebook so IT / hardware can test sample inputs.

