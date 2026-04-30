# Local AI Instructions — Cue

**Purpose:** Workspace-specific context for the Cue repo
**Status:** Canonical
**Scope:** This workspace only
**Created:** 2026-04-28
**Last Updated:** 2026-04-28
**Version:** 1.0.0

All global rules are in `.living-systems/GLOBAL_AI_INSTRUCTIONS.md`. Read that file first. This file adds workspace-specific context only.

---

## Scope

Intelligent clip editor — connected media library and NLE timeline for TKLR content production.

**Workspace code:** CUE
**Decision prefix:** DEC-CUE
**Discussion prefix:** DISC-CUE
branch-strategy: main-only

---

## Key Architectural Rules

- Cue never stores source media files permanently — asset references only
- Playback is always HLS via `hls.js` — no direct file downloads
- Consumer adapters (MAP, Chassis) own all domain-specific translation; Cue core is domain-agnostic
- The timeline data model must support multi-track from day one, even before the Edit screen ships
- Temporary files (proxy segments, export staging) must have automatic cleanup

---

## Integration Contracts

Cue supports two types of consumers:
- **MAP** — pushes F1 moments, clips, HLS manifests, and analysis metadata into the Cue asset library
- **Chassis** — pushes podcast episode recordings, HLS manifests, and transcripts into the Cue asset library

API contracts are versioned. Any breaking change requires a decision doc before implementation.

---

## Enqueueing Blacksmith Tasks

Use the `enqueue_task` MCP tool — never raw SQL INSERTs into `brain.blacksmith_queue`, and never `RemoteTrigger`.

**The brief must be committed and pushed to GitHub before calling `enqueue_task`.** The tool fetches the spec file from GitHub at enqueue time and fails immediately if it isn't found.

If the brief isn't on GitHub yet, pass `spec_content` inline:
```
enqueue_task(task_name=..., spec_path=..., repo=..., spec_content="<brief text>")
```

---

**End Local AI Instructions — Cue**
