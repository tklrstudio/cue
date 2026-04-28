# Decision: Timeline Data Model

**Created:** 2026-04-28
**Decision Status:** Implemented
**Decision ID:** DEC-CUE-2026-04-28-052000
**Workspace:** cue
**Branch:** Operations

---

## Problem

Cue needs a core data model before any application code can be written. The model must support the Cut screen (asset library), the Edit screen (timeline), and the intelligence layer (AI-curated bins) — while remaining domain-agnostic and future-proof for multi-tenancy.

**Background:**
Cue is an intelligent clip editor. Assets are pre-analysed and pre-ingested by consumer systems (MAP, Chassis) before the editor opens. The editor's job is purely creative decision-making — no importing, no file management.

**Current state:**
No schema exists. This is the foundational design decision for the project.

---

## Constitutional Alignment

- [x] **Values** — Creative output (enables faster, higher-quality content production)
- [x] **Temporal** — Active development commitment Q2 2026
- [x] **Contexts** — Creator context: eliminates friction between having media and finishing a piece
- [x] **Foundations** — Resources: keeps storage cost near zero (references only, no media ownership)
- [x] **Modes** — Builder → Publisher: appropriate for current phase
- [x] **Algorithms** — Watched for Infinite Refinement; transitions explicitly parked; bins auto-generated to avoid manual management overhead

---

## Decision

Six tables in the `cue` schema. Two logical layers: the **library** (assets + bins) and the **timeline** (edit decisions). Domain logic and consumer-specific concepts live in JSONB or in consumer adapters — never in column definitions.

### Schema

```sql
-- Library layer

CREATE TABLE cue.assets (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID,                        -- nullable; enforced by policy layer when multi-tenancy ships
    source_system       TEXT NOT NULL,               -- 'map', 'chassis', etc.
    source_id           TEXT NOT NULL,               -- consumer's own identifier
    asset_type          TEXT NOT NULL,               -- 'video', 'audio', 'image'
    hls_manifest_url    TEXT,                        -- null for audio-only and images
    b2_key              TEXT NOT NULL,
    duration_ms         BIGINT,                      -- null for still images
    width               INT,
    height              INT,
    has_audio           BOOLEAN NOT NULL DEFAULT false,
    has_video           BOOLEAN NOT NULL DEFAULT false,
    metadata            JSONB NOT NULL DEFAULT '{}', -- consumer-defined: tags, quality scores, transcripts, race context, etc.
    ingested_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (source_system, source_id)
);

CREATE TABLE cue.bins (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID,
    name                TEXT NOT NULL,
    description         TEXT,
    generated_by        TEXT NOT NULL DEFAULT 'user', -- 'user' | 'ai'
    generation_criteria JSONB,                        -- query/prompt used to populate; enables re-run and refinement
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE cue.bin_assets (
    bin_id      UUID NOT NULL REFERENCES cue.bins(id) ON DELETE CASCADE,
    asset_id    UUID NOT NULL REFERENCES cue.assets(id) ON DELETE CASCADE,
    added_by    TEXT NOT NULL DEFAULT 'user', -- 'user' | 'ai'
    added_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (bin_id, asset_id)
);

-- Timeline layer

CREATE TABLE cue.timelines (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID,
    title           TEXT NOT NULL,
    frame_rate      NUMERIC(8,4) NOT NULL DEFAULT 25.0,
    resolution_w    INT,
    resolution_h    INT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE cue.timeline_clips (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    timeline_id             UUID NOT NULL REFERENCES cue.timelines(id) ON DELETE CASCADE,
    asset_id                UUID NOT NULL REFERENCES cue.assets(id),
    placement_group_id      UUID,        -- links the video + audio rows for the same source clip placement; null for audio-only and image assets
    track_index             INT NOT NULL, -- 0 = V1/A1, 1 = V2/A2, etc.
    track_type              TEXT NOT NULL, -- 'video' | 'audio'
    timeline_position_ms    BIGINT NOT NULL,
    in_point_ms             BIGINT NOT NULL DEFAULT 0,
    out_point_ms            BIGINT NOT NULL,
    cut_type                TEXT NOT NULL DEFAULT 'hard', -- 'hard' | 'j_cut' | 'l_cut'
    -- NOTE: audio_level column was removed by DEC-CUE-2026-04-28-080000.
    -- Audio level (and all transforms) are stored in cue.clip_keyframes instead.
    -- Do NOT add audio_level here when deploying this schema.
    created_at              TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at              TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE cue.timeline_captions (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    timeline_id             UUID NOT NULL REFERENCES cue.timelines(id) ON DELETE CASCADE,
    track_index             INT NOT NULL DEFAULT 0,
    timeline_position_ms    BIGINT NOT NULL,
    duration_ms             BIGINT NOT NULL,
    text                    TEXT NOT NULL,
    style                   JSONB NOT NULL DEFAULT '{}', -- font, size, colour, position, etc.
    created_at              TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at              TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Key Design Decisions

**`tenant_id` on assets, bins, timelines (nullable)**
Single-tenant (TKLR) now — all rows are null. When paid Chassis customers arrive, a policy layer filters by `tenant_id`. MAP assets and Chassis-customer assets are isolated by `tenant_id`, not `source_system`. No migration needed at that point.

**Two rows per clip placement (video + audio)**
Per NLE convention, video and audio occupy separate tracks. A video asset placed on the timeline creates two `timeline_clips` rows: one `track_type='video'` on a video track, one `track_type='audio'` on an audio track. Both share a `placement_group_id` UUID generated at placement time. J/L cuts are the delta between their `timeline_position_ms` values — no separate cut modelling required.

**Audio-only assets: single row**
Audio-only clips (`asset_type='audio'`) create one `timeline_clips` row with `track_type='audio'`. No `placement_group_id`.

**Still images: video track only**
Still images (`asset_type='image'`) create one `timeline_clips` row with `track_type='video'`. `duration_ms` on the asset is the default hold duration; `out_point_ms - in_point_ms` on the clip is the actual hold.

**Captions: separate table**
Captions have no asset reference — they are inline text with timing and style. `timeline_captions` shares the track index model but is structurally distinct from clips.

**Transitions: parked**
Not modelled. `cut_type` on `timeline_clips` is sufficient for hard/J/L. Dissolves, fades, and wipes are out of scope until explicitly needed.

**`metadata` JSONB on assets**
All consumer-specific data (F1 tags, quality scores, transcripts, race context, participant data) lives here. Cue core never queries into it. Consumer adapters write it; consumer UIs may read it for display purposes.

**Bins: global, AI-curated**
Bins are library-level (not timeline-scoped). The intelligence layer generates and maintains them via `generation_criteria` JSONB. Users can edit, rename, merge, or trigger new generation — but never manually assign individual clips. `generated_by` distinguishes AI-created from user-created bins.

**Rationale:**
Clean separation between what the consumer knows (asset metadata) and what the editor knows (edit decisions). The library layer is consumer-owned; the timeline layer is purely Cue. Multi-tenancy is a policy concern, not a schema concern.

**Alternatives considered:**
1. Single `timeline_clips` table with nullable video/audio rows per clip: rejected — conflates two independent track placements into one row, complicates overlap detection and J/L cut logic.
2. `consumer` column on `timelines`: rejected — violates Cue's domain-agnostic principle; a timeline has no consumer identity.
3. Bins scoped per timeline (like Resolve): rejected — Cue's library is shared and pre-populated; global bins allow the intelligence layer to maintain them across all edit sessions.

**Scope:**
This decision defines the complete foundational schema. It does not cover: migration scripts, application models, API contracts, or the Cut/Edit screen UI.

---

## Consequences

### Positive
- Schema supports multi-track, J/L cuts, audio-only, stills, and captions from day one
- `tenant_id` pattern enables multi-tenancy without future migration
- `metadata` JSONB keeps Cue core domain-agnostic
- AI-curated bins eliminate manual library management

### Neutral
- Two rows per video clip placement requires the application layer to manage `placement_group_id` generation at write time

### Risks
- `metadata` JSONB is opaque to Postgres queries — if consumer-specific filtering in the Cut screen becomes important, indexed columns or a `metadata_search` tsvector may be needed later
- `tenant_id` policy enforcement is deferred — must be implemented before any second tenant onboards

---

## Software Delivery

### Schema Impact
Six new tables in `cue` schema. New schema — no migration of existing data. Backwards compatible: yes (greenfield).

### Data Integrity
No existing rows. No backfill required.

### Affected Modules
- All future Cue application modules (asset ingestion, timeline CRUD, Cut screen, Edit screen, render pipeline)
- MAP adapter (writes to `cue.assets`)
- Chassis adapter (writes to `cue.assets`)

### Test Coverage
- Schema integrity: FK constraints, unique constraints, NOT NULL constraints
- Placement group logic: verify two rows share a `placement_group_id` after placement
- Track overlap detection: no two clips on the same track overlap in `[timeline_position_ms, timeline_position_ms + (out_point_ms - in_point_ms))`
- Bin membership: asset can appear in multiple bins; removing a bin does not remove the asset

### Rollback Posture
Greenfield schema — drop the `cue` schema to roll back. No data at risk.

### Environment Notes
Schema lives in the shared PostgreSQL instance used by all TKLR services. Uses `cue` schema namespace to avoid collisions.

---

## Execution Checklist

### 1. Schema / Migration
- [ ] Create `cue` schema
- [ ] Run table creation DDL (in order: assets → bins → bin_assets → timelines → timeline_clips → timeline_captions)
- [ ] Add indexes: `assets(source_system, source_id)`, `assets(tenant_id)`, `timeline_clips(timeline_id)`, `timeline_clips(placement_group_id)`, `bins(tenant_id)`

### 2. Code Changes
- [ ] SQLAlchemy models (or raw DDL module) for all six tables
- [ ] Asset ingestion endpoint (consumers push to `cue.assets`)
- [ ] Timeline CRUD endpoints

### 3. Documentation
- [ ] Update `_context/CHARTER.md` Architecture section with final table list
- [ ] Add GDL-S schema block to any briefs that touch these tables

### 4. AI Usage Guide Required
- [ ] No AI guide required — schema design, no user-facing behaviour change

### 5. Human Usage Guide Required
- [ ] No human guide required at this stage — no UI exists yet

### 6. Change Explainer Required
- [ ] No explainer required — greenfield, no users

### 7. Cross-References and Subsystem Impact
- [ ] MAP adapter will need the asset contract spec before integration
- [ ] Chassis adapter will need the asset contract spec before integration

### 8. Verification
- [ ] All six tables created successfully
- [ ] FK constraints verified
- [ ] Unique constraint on `(source_system, source_id)` verified

---

## Implementation Plan

**Execution environment:** VPS (shared PostgreSQL instance)

**Execution approach:**
1. Create `cue` schema
2. Run DDL for all six tables in dependency order
3. Add indexes
4. Verify constraints

**Estimated effort:** S-tier (schema only, no application code)

**Risk level:** Low — greenfield, no existing data

---

## Problems Solved

No significant problems.

---

## Execution Record

**Executed:** —
**Executed by:** —
**Execution time:** —
**Result:** —

---

## Related

**Goals:**
- Cue active development Q2 2026 (primary)

**Decisions:**
- Supersedes: none (first decision)
- **Partially superseded by:** DEC-CUE-2026-04-28-080000 — removes `audio_level` from `timeline_clips` and introduces `cue.clip_keyframes` for all per-clip property values (transform + audio)

---

**End of Decision Record**
