# M1 Local EMS Interface

M1 is the first local-control interface for a BESS + EV charging site.

Scope: Import Only, no PV, no export, no V2G.

This repository defines the interface contract between cloud planning, local EMS decision logic, and IT/hardware integration. It is not a final optimization algorithm or production dispatch controller.

## Core Idea

```text
L3 cloud parameter packet + L1 safety limits + L2 local realtime state
        ↓
local formula inside safety boundaries
        ↓
Y_t = { mode, p_ev_limit_kw, p_bess_target_kw, reason_code }
```

## Read This First

| Audience | Document |
|---|---|
| Latest algorithm review context | `algorithm-context/2026-05-28-m1-edge-control/README.md` |
| Plain-language v0.3 explanation | `algorithm-context/2026-05-28-m1-edge-control/m1-model1-explainer-v0.3.md` |
| Current v0.3 formula | `algorithm-context/2026-05-28-m1-edge-control/m1-model1-local-algorithm-v0.3.md` |
| Product / cloud alignment | `docs/m1-three-tables-for-alignment.md` |
| IT interface handoff | `docs/m1-local-control-interface-v0.1.md` |
| Technical appendix | `docs/m1-bottom-up-interface-plan.md` |
| Meeting context | `meeting-notes/2026-05-27-import-only-parameterization-summary.md` |

## Repository Layout

| Folder | Content |
|---|---|
| `docs/` | Current M1 handoff documents |
| `algorithm-context/` | Latest algorithm-debugging context, review briefs, and notebooks |
| `work-notes/` | Earlier working notes and expert synthesis drafts |
| `meeting-notes/` | Meeting summaries and alignment notes |
| `source-materials/` | Original PPT/PDF/screenshot materials |
| `references/` | Literature registry and source PDFs used in prior EV/BESS work |

## M1 Scope

M1 means:

- BESS + EV charging only
- Import Only
- No PV
- No export
- No V2G

## Included

- Cloud half-hour parameter package
- Local realtime state
- Hard constraints
- Mode / target / reason_code output
- Fallback and protect states
- M1 test matrix
- Meeting context and working notes
- Original source PPT/PDF materials
- Literature registry and source PDFs

## Excluded

- PV dispatch
- Export / sell-back
- V2G
- Full EMS optimization
- Per-gun EV allocation
- LLM in realtime control

## Decision Contract

```text
Y_t = {
  mode,
  p_ev_limit_kw,
  p_bess_target_kw,
  reason_code
}
```

Output modes:

```text
NORMAL
BESS_SUPPORT
EV_LIMIT
BESS_CHARGE
SAFE_PROTECT
```

Example:

```text
mode = BESS_CHARGE
p_ev_limit_kw = 120
p_bess_target_kw = 30
reason_code = SOC_LOW
```

## Validation

Each test case should specify:

- input state
- triggered constraint
- expected mode
- expected target
- expected reason_code

## Open Questions

- Which fields are already available locally?
- What are the refresh frequency, latency, and timestamp guarantees?
- How should `low / medium / high` map to lower-level kW or PCS commands?
- Can IT provide a mock/replay test environment for M1?
