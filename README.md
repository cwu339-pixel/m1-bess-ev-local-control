# M1 Local EMS Interface

M1 is the first local-control interface for a BESS + EV charging site.

Scope: Import Only, no PV, no export, no V2G.

This repository defines the interface contract between cloud planning, local EMS decision logic, and IT/hardware integration. It is not a final optimization algorithm or production dispatch controller.

## Core Idea

```text
cloud half-hour parameter packet + local realtime state
        ↓
hard constraints filter infeasible actions
        ↓
decision = { mode, target, reason_code }
```

## Read This First

| Audience | Document |
|---|---|
| Latest algorithm review context | `algorithm-context/2026-05-28-m1-edge-control/README.md` |
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
- Full economic optimization
- Final kW-level dispatch
- LLM in realtime control

## Decision Contract

```text
decision = {
  mode,
  target,
  reason_code
}
```

Output modes:

```text
BESS_CHARGE_TO_SOC
BESS_DISCHARGE_TO_EV
BESS_HOLD
EV_LIMIT
SAFE_PROTECT
SAFE_FALLBACK
```

Example:

```text
mode = BESS_DISCHARGE_TO_EV
target = medium
reason_code = EV_DEMAND_HIGH
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
