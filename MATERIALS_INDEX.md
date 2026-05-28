# Materials Index

This repository collects both the current M1 handoff and the supporting materials used to get there.

## Current Handoff

| File | Purpose |
|---|---|
| `README.md` | Repository entry point |
| `algorithm-context/2026-05-28-m1-edge-control/README.md` | Latest algorithm context pack for the M1 edge-control review |
| `algorithm-context/2026-05-28-m1-edge-control/review-brief-for-gpt-5-5-pro.md` | Review brief for improving the formula with SOC bands and price ranks |
| `docs/m1-three-tables-for-alignment.md` | Short alignment version: problem definition, constraints, parameter list |
| `docs/m1-local-control-interface-v0.1.md` | Main M1 local-control interface handoff |
| `docs/m1-bottom-up-interface-plan.md` | Technical appendix for bottom-up interface and test plan |

## Latest Algorithm Context

| File | Purpose |
|---|---|
| `algorithm-context/2026-05-28-m1-edge-control/meeting-summary-2026-05-28.md` | Meeting summary and latest decision boundary |
| `algorithm-context/2026-05-28-m1-edge-control/m1-four-control-loops-v0.1.md` | Four-loop explanation: L1 protection, power aggregation, economic dispatch, gun/module allocation |
| `algorithm-context/2026-05-28-m1-edge-control/m1-local-algorithm-brief-v0.2.md` | Latest reviewed M1 local algorithm draft with price ranks and physical-cap fixes |
| `algorithm-context/2026-05-28-m1-edge-control/m1-local-algorithm-brief-v0.1.md` | Current local-algorithm explanation with formulas and flow |
| `algorithm-context/2026-05-28-m1-edge-control/m1-formula-algorithm-v0.1.md` | Formula-focused draft before price-ranking integration |
| `algorithm-context/2026-05-28-m1-edge-control/m1-x-fx-y-deliverable-v0.1.md` | x -> f(x) -> y handoff draft |
| `algorithm-context/2026-05-28-m1-edge-control/m1-data-requirements-for-jin-v0.1.md` | IT / Jin data-requirement draft |
| `algorithm-context/2026-05-28-m1-edge-control/notebooks/m1-mvp-local-algorithm-test.ipynb` | Runnable notebook for current no-price v0.1 formula |
| `algorithm-context/2026-05-28-m1-edge-control/source-notes/ning-edge-control-strategy-summary.md` | Summary of Ning's edge-control draft and how it maps to M1 |
| `algorithm-context/2026-05-28-m1-edge-control/source-notes/ning-edge-control-strategy-draft-02.txt` | Latest Ning edge-control draft; supersedes draft 01 for formula review |
| `algorithm-context/2026-05-28-m1-edge-control/source-notes/ning-edge-control-strategy-draft-02-update-note.md` | Short note on the draft 02 correction |
| `algorithm-context/2026-05-28-m1-edge-control/source-notes/ning-edge-control-strategy-draft-01.txt` | Ning edge-control draft converted from docx |
| `algorithm-context/2026-05-28-m1-edge-control/source-notes/control-logic-v0.3-ppt-extracted.md` | Extracted text from the control-logic PPT |

## Meeting Notes

| File | Purpose |
|---|---|
| `meeting-notes/2026-05-27-import-only-parameterization-summary.md` | Meeting context for Import Only parameterization, IT interface, testing, and Perfect/MLflow discussion |

## Source Materials

| File | Purpose |
|---|---|
| `source-materials/控制逻辑_技术交流_v0.3.pptx` | Original control-logic PPT |
| `source-materials/report_PV_BESS_20251210.pdf` | PV/BESS related report |
| `source-materials/screenshots/12-models-import-only-discussion.png` | Screenshot of 12-control-model discussion |
| `source-materials/screenshots/math-framework-checklist.png` | Screenshot of math-framework checklist |

## Working Notes

| File | Purpose |
|---|---|
| `work-notes/2026-05-25-pv-bess-ppt-page-notes.md` | Page-by-page PPT explanation |
| `work-notes/2026-05-25-pv-bess-local-control-logic-v1.md` | V1 local-control logic explanation |
| `work-notes/2026-05-25-pv-bess-local-control-logic-v2-expert.md` | Expert-style V2 local-control logic |
| `work-notes/2026-05-25-first-scenario-load-ev-bess-local-logic.md` | First scenario: load + EV + BESS local logic |
| `work-notes/2026-05-25-local-algorithm-dimensions-expert-synthesis.md` | Expert synthesis of local algorithm dimensions |
| `work-notes/2026-05-25-local-algorithm-optimization-dimensions.md` | Broader optimization-dimension draft |
| `work-notes/2026-05-26-local-algorithm-clean-summary.md` | Clean summary of local algorithm discussion |
| `work-notes/2026-05-26-local-algorithm-meeting-prep.md` | Meeting preparation notes |

## References

| File / Folder | Purpose |
|---|---|
| `references/ev-charging-literature-registry-2026-05-15.md` | Literature registry from earlier EV charging research |
| `references/session-source-ledger-2026-05-15.md` | Source ledger |
| `references/2026-05-14-stage0-ppt-reference-pack.md` | Reference pack for prior Stage-0 materials |
| `references/source_pdfs/` | Local copies of supporting reference PDFs |

## Scope Note

The active implementation scope is still M1:

```text
BESS + EV
Import Only
No PV
No export
No V2G
```

The additional materials are included for context and traceability. They do not expand the current M1 execution scope.
