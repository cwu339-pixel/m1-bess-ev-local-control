# Ning Edge-Control Draft 02 Update Note

Source:

```text
source-notes/ning-edge-control-strategy-draft-02.txt
```

Draft 02 should supersede draft 01 for review purposes.

## What Changed From Draft 01

Draft 01 treated MIC headroom and the price reward as additive terms:

```text
charge_kw = alpha * f1[band] + beta_max[band] * (1 - price_rank)
```

The issue is that the price reward can add extra kW outside the MIC headroom budget.

Draft 02 changes the structure:

```text
alpha = max(0, (MIC - siteload_smooth) * (1 - margin))
u = f1[band] + g[band] * (1 - price_rank)
charge_kw = alpha * u
```

with:

```text
0 <= u <= 1
f1[band] + g[band] <= 1
```

So MIC headroom is now the multiplicative cap. Price only changes the utilization ratio of available headroom.

## Why It Matters For M1

For M1, price logic should not create extra power outside physical limits.

When adapting this into the BESS + EV formula, use the same pattern:

```text
physical_cap -> utilization_ratio -> target_power
```

Do not use:

```text
physical_cap + price_bonus_kw
```

M1 still differs from Ning's draft because M1 needs:

```python
Y_t = {
    "mode": "...",
    "p_ev_limit_kw": 0.0,
    "p_bess_target_kw": 0.0,
    "reason_code": "...",
}
```

Ning's draft is charge-only and outputs only `charge_kw`.

