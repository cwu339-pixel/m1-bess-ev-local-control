# Ning Edge-Control Strategy Draft Summary

Source file:

```text
source-notes/ning-edge-control-strategy-draft-01.txt
```

## What This Draft Is

This draft is a reference design for an edge battery charging controller.

Its main idea:

```text
Cloud thinks.
Edge executes.
Safety is independent.
```

The edge controller should run simple arithmetic, not a full optimizer.

## What It Outputs

The draft outputs only:

```text
charge_kw
```

Meaning:

```text
how much power to charge the battery from the grid right now
```

It is not the same as the current M1 output, because M1 also needs EV power limiting and possible BESS support for EV demand.

## Key Formula Idea

The draft uses:

```text
alpha = max(0, (MIC - siteload_smooth) * (1 - margin))
```

Then it chooses charging power by SOC band:

```text
SOC < p10  -> emergency charge
SOC < p25  -> catch up charge
SOC < p50  -> moderate charge
SOC < p75  -> light charge
SOC >= p75 -> no charge
```

It adds price sensitivity through:

```text
beta_max[band] * (1 - price_rank[t])
```

So:

```text
price_rank = 0 -> cheap -> more charge
price_rank = 1 -> expensive -> less charge
```

## What We Should Reuse

For M1, reuse these ideas:

1. Cloud should calculate SOC bands.
2. Cloud should calculate price ranks.
3. Edge should consume simple parameters, not raw tariff tables.
4. Safety should stay separate from strategy.
5. Edge should use simple formulas and clips, not a large optimizer.

## What We Should Not Copy Directly

The draft is "charge-only". It says discharge is passive.

M1 is different:

```text
BESS + EV
Import Only
EV pool limit required
BESS target required
```

So the draft cannot be copied directly.

M1 must output:

```python
Y_t = {
    "mode": "...",
    "p_ev_limit_kw": 0.0,
    "p_bess_target_kw": 0.0,
    "reason_code": "...",
}
```

## Practical Translation For M1

Use Ning's draft as a design pattern:

```text
SOC band + price rank + MIC headroom
```

but adapt it to M1:

```text
EV gap + BESS support + EV limit + optional BESS charge
```

