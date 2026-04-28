# Cue Charter

**Purpose:** Intelligent clip editor — pre-analysed media library and NLE timeline for TKLR content production
**Status:** Canonical
**Scope:** Project
**Branch:** Operations
**Workspace Code:** CUE
**Type:** Software
**Created:** 2026-04-28
**Last Updated:** 2026-04-28
**Version:** 1.0.0

---

## Purpose

Cue exists to eliminate the gap between having media and having a finished piece. Every asset in the library is already ingested, analysed, tagged, and understood before the editor opens — so the editor is purely about creative decisions, not discovery or file management.

---

## Constitutional Alignment

**Context:** Creator
**Values served:** Creative output, sustainable systems
**Foundation developed:** Competencies, Resources
**Mode:** Builder → Publisher
**Current commitment:** Medium — active development Q2 2026

---

## Success Criteria

- An editor can open Cue and see a populated, searchable library of assets without importing anything
- A Short, podcast episode, or long-form package can be assembled and exported from a single interface
- Chassis and MAP each integrate via defined API contracts — neither requires changes to Cue core
- Export renders from source assets (via B2/HLS) without Cue storing the media permanently
- A new consumer (beyond MAP and Chassis) can integrate by implementing the published asset contract

---

## Boundaries

What Cue is **NOT**:

- **Not a media store** — Cue stores asset references and timeline metadata. It never owns source files permanently. Temporary storage (HLS proxy segments, export staging) must auto-clean.
- **Not a file manager** — No import dialogs, no project folders, no media bins to organise manually. The library is always pre-populated by connected systems.
- **Not motorsport-specific** — Cue is the editor. MAP and Chassis are the consumers. The core has no F1, podcast, or TKLR-specific concepts.
- **Not a DaVinci Resolve replacement for arbitrary workflows** — Cue serves a defined set of consumers with pre-analysed media. General-purpose NLE features are out of scope.
- **Not a publishing tool** — Cue produces the edit. Publishing to YouTube, Spotify, or social platforms is Chassis Publisher's job.

---

## Design Principles

In priority order:

1. **Intelligence before interaction** — Everything that can be known about a clip is known before the editor opens. The editing session is purely creative decision-making.
2. **References, not copies** — Media lives in B2 (or wherever consumers store it). Cue holds pointers. This keeps storage costs near zero and eliminates sync problems.
3. **HLS for playback** — All media is served via HLS. The browser fetches only the segments it needs. No local proxy generation required.
4. **Consumer adapters own the translation** — MAP and Chassis each have an adapter that translates their data model into Cue's asset schema. Cue core knows nothing about moments, episodes, or race data.
5. **Edit-compatible data model from day one** — Even before the Edit screen ships, the timeline schema supports multi-track, split audio/video, and J/L cuts. The UI is phased; the model is not.

---

## Architecture

Cue is a Python + Starlette web application backed by PostgreSQL, with Bouncer handling authentication. The UI is vanilla HTML/CSS/JS with `hls.js` for media playback.

**Note:** The Cut + Edit timeline UI is interaction-heavy. If vanilla JS proves insufficient for timeline rendering at the required fidelity, a minimal JS framework (Lit or similar) may be introduced via a decision doc.

```
Cue (editor)
├── Asset Library — references to media from connected consumers
├── Timeline — edit decisions: in/out points, track layout, cut types, audio levels
├── Cut Screen — select and sequence from the asset library
├── Edit Screen — multi-track, audio, J/L cuts (phased after Cut screen)
├── Render — ffmpeg-based export, pulls from B2, outputs to B2
└── Integrations
    ├── MAP Adapter — moments, clips, race data → Cue asset schema
    └── Chassis Adapter — episodes, recordings, transcripts → Cue asset schema
```

**Playback stack:** B2 serves HLS segments + `.m3u8` manifests → `hls.js` in browser → Cue timeline player controls seek/in/out.

**Timeline data model:** Two layers:
1. **Asset references** — B2 key, HLS manifest URL, source system, all analysis metadata (tags, timestamps, quality scores, transcripts) passed in by the consumer adapter
2. **Edit decisions** — in/out points, track placement, cut type (hard/J/L), audio levels, transitions — owned entirely by Cue

---

## Integration Ecosystem

| Consumer | What it provides | Status |
|----------|-----------------|--------|
| MAP | F1 moments, clips, HLS manifests, race context, analysis metadata | Planned |
| Chassis | Podcast episode recordings, HLS manifests, transcripts, participant data | Planned |

New consumers must implement the published Cue asset contract. They do not require changes to Cue core.

---

## API Contracts

Cue exposes HTTP APIs for:
- Asset ingestion (consumers push assets into the library)
- Timeline CRUD (create, read, update, delete edit sessions)
- Export trigger (kick off a render job)
- Export status/webhook (render completion callback)

Consumers implement:
- Asset push (translating their data model into Cue's asset schema)

Contracts are versioned. Breaking changes require a decision doc.

---

## Security Requirements

**Authentication:** Yes — Bouncer (`cue_session` cookie). MFA enforced. Required on all user-facing routes.

---

## Architectural Invariants

- Source media is never stored permanently by Cue
- Playback is always via HLS — no direct file downloads for editing
- Consumer adapters own all domain-specific translation; Cue core is domain-agnostic
- The timeline data model must support multi-track from day one, regardless of UI phase
- Temporary files (proxy segments, export staging) must have automatic cleanup

---

## Binding Rules

- Consumer systems push assets to Cue — Cue never pulls or discovers media
- API contracts must be documented before any consumer ships an integration
- Export renders pull directly from B2 via source HLS — no re-encoding of already-encoded assets unless required
- Core must function (asset library, timeline editing) without any consumer connected

---

## Failure Patterns to Watch

- **Scope creep toward MAP** — Cue must not absorb MAP's moment analysis or race data concepts. Keep the boundary clean.
- **Media storage creep** — Temporary files accumulating and never being cleaned. Audit cleanup on every render path.
- **Consumer coupling** — If adding a new consumer requires changes to Cue core, the adapter boundary has been violated.
- **Premature Edit screen** — Build Cut screen until it's proven. Resist building Edit screen complexity before it's needed.
- **Framework sprawl** — If a JS framework is introduced, it must be via a decision doc. Don't let it creep in as a convenience.

---

## Guiding Principle

*The edit begins when the intelligence ends.*

---

*Review this charter when scope, architecture, or integration contracts change materially.*
