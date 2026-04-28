# Build Tracking — Estimate vs Actual

**Purpose:** Track CC build estimates against actuals as each layer ships
**Status:** Canonical
**Scope:** Project
**Created:** 2026-04-28
**Last Updated:** 2026-04-28

---

## Methodology

- **Estimate basis:** Observed Blacksmith task velocity (~5–9 min/task, ~$1–3/task on Sonnet 4.6). See `_context/CHARTER.md` for the full estimate rationale.
- **Actuals:** Record task count, wall-clock calendar time, and cost from `brain.task_runs` when each layer completes.
- **Calendar time** = date first task fired → date last task completed for that layer (includes gaps between sessions, not just compute time).
- **Cost** = sum of `cost_usd_estimate` from `brain.task_runs` for all tasks in the layer.

---

## Layer Tracking

### 1. Schema + Models + CRUD

| | Estimate | Actual |
|-|----------|--------|
| Tasks | 4–6 | — |
| Calendar time | ~0.5 days | — |
| Cost (USD) | ~$5–15 | — |
| Started | — | — |
| Completed | — | — |
| Notes | — | — |

---

### 2. Cut Screen

| | Estimate | Actual |
|-|----------|--------|
| Tasks | 5–8 | — |
| Calendar time | ~1 day | — |
| Cost (USD) | ~$10–20 | — |
| Started | — | — |
| Completed | — | — |
| Notes | — | — |

---

### 3. Edit Screen — Timeline Canvas

| | Estimate | Actual |
|-|----------|--------|
| Tasks | 10–15 | — |
| Calendar time | 2–3 days | — |
| Cost (USD) | ~$25–45 | — |
| Started | — | — |
| Completed | — | — |
| Notes | — | — |

---

### 4. Keyframes UI + Speed Editor Phase 1

| | Estimate | Actual |
|-|----------|--------|
| Tasks | 3–4 | — |
| Calendar time | ~0.5 days | — |
| Cost (USD) | ~$5–10 | — |
| Started | — | — |
| Completed | — | — |
| Notes | — | — |

---

### 5. Render Pipeline

| | Estimate | Actual |
|-|----------|--------|
| Tasks | 4–6 | — |
| Calendar time | ~1 day | — |
| Cost (USD) | ~$10–20 | — |
| Started | — | — |
| Completed | — | — |
| Notes | — | — |

---

## Totals

| | Estimate | Actual |
|-|----------|--------|
| Total tasks | 26–39 | — |
| Total calendar time | ~5–6 days | — |
| Total cost (USD) | $80–150 | — |
| Build started | — | — |
| Build complete | — | — |

---

## Variance Log

Record significant deviations from estimate here as they occur.

| Date | Layer | What happened | Impact |
|------|-------|--------------|--------|
| — | — | — | — |

---

**End Build Tracking**
