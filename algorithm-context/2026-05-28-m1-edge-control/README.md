# M1 Edge Control Algorithm Context Pack

Date: 2026-05-28

This folder collects the latest M1 local-control algorithm context for review.

## One-Sentence Goal

Define a small but correct M1 local economic-dispatch formula for:

```text
BESS + EV
Import Only
No PV
No export
No V2G
```

The goal is not a broad product MVP. The scope is narrow, but the formula and interface should be precise enough for IT / hardware integration tests.

## Current Boundary

We own the local economic-dispatch layer:

```text
SOC band + price signal + MIC headroom + EV/AC aggregate load
        -> p_ev_limit_kw + p_bess_target_kw
```

We do not own these layers in this version:

| Layer | Owner / Source | Notes |
|---|---|---|
| L1 protection | Existing MIC / BMS / PCS / EMS protection | We consume status and limits; we do not reimplement protection. |
| Gun allocation | IT / charger / Youlite-side EV power allocation | We output only site-level `p_ev_limit_kw`, equivalent to `P_gun_pool_max`. |
| Raw price calculation | Cloud | Edge should consume cloud-derived price ranks, not raw tariff tables. |
| Export / sell-back | Out of M1 scope | M1 is Import Only. |

## Fixed M1 Output

```python
Y_t = {
    "mode": "...",
    "p_ev_limit_kw": 0.0,
    "p_bess_target_kw": 0.0,
    "reason_code": "...",
}
```

Interpretation:

| Field | Meaning |
|---|---|
| `mode` | `NORMAL / BESS_SUPPORT / EV_LIMIT / BESS_CHARGE / SAFE_PROTECT` |
| `p_ev_limit_kw` | Site-level EV pool max power, maps to `P_gun_pool_max` |
| `p_bess_target_kw` | Positive = charge, negative = discharge, zero = idle |
| `reason_code` | Main reason for the decision |

## Important Shift After 2026-05-28 Meeting

Earlier drafts used the word "MVP". The intended meaning is not "rough prototype".

The corrected framing is:

```text
Small scope, precise formula.
```

Ning's feedback:

- The current problem definition is acceptable.
- The solution needs to include price ranking.
- L1 protection and gun allocation should be documented as separate loops, but not solved in this algorithm layer.
- IT is mainly waiting for the formula and data-field requirements.
- The next step is to prepare for integration testing, so the formula must be structured and testable.

## What To Read

| File | Purpose |
|---|---|
| `review-brief-for-gpt-5-5-pro.md` | Start here. Contains the review task and current open questions. |
| `meeting-summary-2026-05-28.md` | Plain-language meeting summary and latest decisions. |
| `m1-local-algorithm-brief-v0.2.md` | Latest reviewed M1 formula draft with SOC bands, price ranks, physical caps, and post-slew EV limit. |
| `m1-local-algorithm-brief-v0.1.md` | Current simple local-algorithm explanation with formulas and flow. |
| `m1-x-fx-y-deliverable-v0.1.md` | Earlier x -> f(x) -> y handoff draft. |
| `m1-formula-algorithm-v0.1.md` | Formula-focused draft before price-ranking integration. |
| `m1-data-requirements-for-jin-v0.1.md` | Data-field request draft for IT / Jin. |
| `notebooks/m1-mvp-local-algorithm-test.ipynb` | Runnable notebook implementing the current no-price v0.1 formula. |
| `source-notes/ning-edge-control-strategy-draft-02.txt` | Latest Ning edge-control draft. Use this instead of draft 01 for formula review. |
| `source-notes/ning-edge-control-strategy-draft-02-update-note.md` | Short note on what changed in draft 02 and why it matters for M1. |
| `source-notes/ning-edge-control-strategy-draft-01.txt` | Ning's edge-control strategy draft, converted from docx. |
| `source-notes/ning-edge-control-strategy-summary.md` | Summary of what can and cannot be reused from Ning's draft. |
| `source-notes/control-logic-v0.3-ppt-extracted.md` | Extracted text from the earlier control-logic PPT. |

## Current Missing Piece

The next version should integrate price ranking into the M1 formula.

Important update: Ning's draft 02 changes the price logic from an additive price-bonus term into a utilization-ratio term under the MIC headroom cap. Reviewers should read draft 02 before proposing M1 v0.2.

Recommended price signal boundary:

```text
Cloud computes:
  grid_buy_price_rank_t
  ev_charge_price_rank_t
  spread_rank_t

Edge consumes those ranks.
Edge does not calculate raw tariffs.
```

M1 currently excludes:

```text
export_sell_price
```

because M1 has no export and no V2G.
