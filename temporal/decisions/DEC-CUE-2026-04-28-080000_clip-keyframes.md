# Decision: Clip Keyframes — Transform and Audio

**Created:** 2026-04-28
**Decision Status:** Approved
**Decision ID:** DEC-CUE-2026-04-28-080000
**Workspace:** cue
**Branch:** Operations

---

## Problem

`cue.timeline_clips` has no way to express per-clip transform (position, rotation, scale) or variable audio level. The `audio_level` scalar added in DEC-CUE-2026-04-28-052000 covers the single-value case but cannot represent changes over time, and there is no column at all for spatial transforms.

**Background:**
The Edit screen will need to composite video and image clips at arbitrary positions, scales, and rotations within the canvas — and to set per-clip audio levels. Even before animation is required, a data model that handles "static" values via single keyframes is preferable to adding scalar columns per property, because adding animation later then costs zero schema change.

**Current state:**
`timeline_clips.audio_level NUMERIC(4,3)` exists but is not yet deployed (DEC-CUE-2026-04-28-052000 execution checklist is unchecked). No transform properties exist anywhere.

---

## Constitutional Alignment

- [x] **Values** — Creative output; reduces friction in the production pipeline
- [x] **Temporal** — Cue active development Q2 2026
- [x] **Contexts** — Creator context: enables precise clip positioning and mix control without rebuilding the data layer
- [x] **Modes** — Builder → Publisher: phase-appropriate; scoped strictly to what is needed now
- [x] **Algorithms** — Watched for Infinite Refinement: animation interpolation is explicitly out of scope; only the data structure is decided here

---

## Decision

Add a single `cue.clip_keyframes` table. Remove `audio_level` from `cue.timeline_clips`. A single keyframe at `time_ms = 0` is the canonical way to express a static (non-animated) property value.

### DDL

```sql
-- Remove audio_level from timeline_clips
-- (apply this BEFORE deploying DEC-CUE-2026-04-28-052000 if schema is still undeployed,
--  or as ALTER TABLE if already deployed)
ALTER TABLE cue.timeline_clips DROP COLUMN IF EXISTS audio_level;

CREATE TABLE cue.clip_keyframes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    clip_id         UUID NOT NULL REFERENCES cue.timeline_clips(id) ON DELETE CASCADE,
    property        TEXT NOT NULL,
    -- Allowed values:
    --   'position_x'   pixels from canvas centre (positive = right)
    --   'position_y'   pixels from canvas centre (positive = down)
    --   'scale_x'      multiplier, 1.0 = 100%
    --   'scale_y'      multiplier, 1.0 = 100%
    --   'rotation'     degrees clockwise, 0.0 = upright
    --   'audio_level'  linear multiplier, 1.0 = 0 dBFS, 0.0 = silence
    time_ms         BIGINT NOT NULL,
    -- Clip-relative: 0 = the moment this clip starts on the timeline.
    -- Must be >= 0 and <= (clip.out_point_ms - clip.in_point_ms).
    -- Application layer enforces this range.
    value           NUMERIC NOT NULL,
    interpolation   TEXT NOT NULL DEFAULT 'hold',
    -- 'hold'   — value is constant until the next keyframe (use this now)
    -- 'linear' — linearly interpolated to next keyframe (implement when animation lands)
    -- 'bezier' — cubic bezier easing (future)
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (clip_id, property, time_ms)
);

CREATE INDEX clip_keyframes_clip_id_idx ON cue.clip_keyframes (clip_id);
CREATE INDEX clip_keyframes_clip_property_idx ON cue.clip_keyframes (clip_id, property);
```

### Property identity values (defaults when no keyframe exists)

| Property    | Identity value | Notes                                        |
|-------------|----------------|----------------------------------------------|
| `position_x`| `0`            | Centred horizontally on canvas               |
| `position_y`| `0`            | Centred vertically on canvas                 |
| `scale_x`   | `1.0`          | 100% — no scaling                           |
| `scale_y`   | `1.0`          | 100% — no scaling                           |
| `rotation`  | `0.0`          | No rotation                                  |
| `audio_level`| `1.0`         | Full volume; above 1.0 is a gain boost       |

No keyframe row = use the identity value. The application layer never writes identity-value keyframes unless the user explicitly sets them.

### How to apply keyframes at render / playback

For each property needed at playhead position `P` on the timeline:

1. Compute clip-relative time: `t = P - clip.timeline_position_ms`
2. Query: `SELECT value FROM cue.clip_keyframes WHERE clip_id = $1 AND property = $2 AND time_ms <= $3 ORDER BY time_ms DESC LIMIT 1`
3. If no row → use the identity value for that property
4. If `interpolation = 'hold'` (all cases for now) → use the returned `value` directly
5. `interpolation = 'linear'` (future): fetch the next keyframe and lerp

This query runs at render time for each property × each clip visible at `P`. With the index on `(clip_id, property)` this is fast even for clips with many keyframes.

### Placement convention — what gets written when a clip is placed on the timeline

Nothing is written to `clip_keyframes` at placement time. The application relies on identity-value defaults. Keyframe rows are only created when the user explicitly sets a property value in the Edit screen.

**Exception:** if the user sets a non-identity value for a property and later deletes it via the UI, the row must be deleted (not set to the identity value). Keeping identity-value rows wastes space and creates ambiguity.

### Uniform scale (UI convention, not a schema concern)

The table stores `scale_x` and `scale_y` independently. When the Edit screen shows a single "Scale" handle with a lock toggle, it writes to both properties at the same `time_ms`. The lock is UI state only — the database has no concept of it.

### Audio — `track_type = 'video'` clips with audio

A `timeline_clips` row with `track_type = 'video'` that belongs to a video asset (`has_audio = true`) gets its own audio level. Its paired audio row (`track_type = 'audio'`, same `placement_group_id`) gets its own level too. They are independent — this is correct: the Edit screen may want to trim the embedded audio separately from the audio track.

### What this decision does NOT cover

- Interpolation between keyframes (implement when animation lands)
- Bezier handle coordinates for easing curves
- Per-property mute / bypass toggle
- Opacity (a future property; add a row to `clip_keyframes` with `property = 'opacity'` and `value 0.0–1.0` when needed — no schema change required)
- Clip effects / colour grading (separate concern)

---

## Consequences

### Positive
- Animation is a zero-schema-change addition: just write more keyframe rows and implement interpolation in the render path
- `clip_keyframes` handles all future properties without migration (new `property` values are additive)
- Cascade delete on `clip_id` keeps the table clean when clips are removed
- No ambiguity between a scalar column and keyframe rows for the same property

### Neutral
- Render path must now query `clip_keyframes` for every property; previously `audio_level` was on the row itself. The index makes this fast.
- Application layer must handle the "no row = identity" convention consistently

### Risks
- `property` is free-text — typos create silent bugs. Application layer must validate against the allowed set before INSERT. A Postgres CHECK constraint is an alternative but makes adding properties require a migration.
- `time_ms` range enforcement (>= 0, <= clip duration) is application-layer only; out-of-range keyframes are silently ignored at render time

---

## Software Delivery

### Schema Impact
- **New table:** `cue.clip_keyframes` (see DDL above)
- **Removed column:** `cue.timeline_clips.audio_level`
- Migration required: yes (DROP COLUMN; CREATE TABLE)
- Backwards compatible: yes if applied before DEC-CUE-2026-04-28-052000 is deployed (schema is still greenfield); DROP COLUMN needed otherwise

### Data Integrity
If `timeline_clips` is already deployed with `audio_level`:
- Back up existing `audio_level` values
- For each row with `audio_level != 1.0`, insert a `clip_keyframes` row: `property='audio_level', time_ms=0, value=audio_level`
- Then DROP COLUMN

If schema is not yet deployed (current state as of 2026-04-28): just omit `audio_level` from the CREATE TABLE DDL in DEC-CUE-2026-04-28-052000 and apply this DDL after.

### Affected Modules
- `cue.timeline_clips` DDL (remove `audio_level` column)
- Any ORM model / SQLAlchemy class for `timeline_clips`
- Render / export pipeline (read keyframes instead of `audio_level` scalar)
- Edit screen UI (keyframe panel, transform controls, audio level fader)
- Timeline CRUD API (clip creation no longer accepts `audio_level`)

### Test Coverage
- Insert a clip with no keyframes; verify render path returns identity values for all properties
- Insert a keyframe at `time_ms=0` for `audio_level=0.5`; verify render path at `t=0` returns `0.5`
- Insert two `hold` keyframes at `t=0` (value 0.5) and `t=1000` (value 0.8); verify render at `t=500` returns `0.5` and render at `t=1500` returns `0.8`
- Delete a clip; verify cascade delete removes all its keyframes
- Attempt to insert duplicate `(clip_id, property, time_ms)`; verify unique constraint fires
- Verify `audio_level` column no longer exists on `timeline_clips`

### Rollback Posture
Before any data: drop `cue.clip_keyframes`, re-add `audio_level` to `timeline_clips`. No data loss.
After data is written: export keyframe rows for `audio_level` back to the column before dropping the table.

### Environment Notes
Same shared PostgreSQL instance as all other Cue tables. Apply in a single transaction: `ALTER TABLE … DROP COLUMN` + `CREATE TABLE clip_keyframes` together.

---

## Execution Checklist

### 1. Schema / Migration
- [ ] If `timeline_clips` already deployed: run `ALTER TABLE cue.timeline_clips DROP COLUMN IF EXISTS audio_level;`
- [ ] If `timeline_clips` not yet deployed: remove `audio_level` column from DEC-CUE-2026-04-28-052000 DDL before running it
- [ ] Run `CREATE TABLE cue.clip_keyframes` DDL (above)
- [ ] Run `CREATE INDEX clip_keyframes_clip_id_idx`
- [ ] Run `CREATE INDEX clip_keyframes_clip_property_idx`
- [ ] Verify table exists with correct columns: `\d cue.clip_keyframes`
- [ ] Verify unique constraint: attempt duplicate insert and confirm error
- [ ] Verify cascade: delete a clip and confirm its keyframe rows are gone

### 2. Code Changes
- [ ] Update SQLAlchemy (or ORM) model for `timeline_clips` — remove `audio_level` field
- [ ] Add SQLAlchemy model for `clip_keyframes` with all six `property` values as a validated enum/constant set
- [ ] Add `get_keyframe_value(clip_id, property, time_ms)` helper in the render/playback layer — returns identity value when no row found
- [ ] Update clip creation endpoint: remove `audio_level` from request body / response
- [ ] Validate `property` against allowed set on INSERT in API layer

### 3. Documentation
- [ ] Update DEC-CUE-2026-04-28-052000 to note `audio_level` column is superseded by this decision (note added in that doc)
- [ ] Update `_context/CHARTER.md` Architecture section with `clip_keyframes` table

### 4. AI Usage Guide Required
- [ ] No AI guide required — internal data model, no user-facing AI behaviour change

### 5. Human Usage Guide Required
- [ ] No human guide required at this stage — no UI exists yet

### 6. Change Explainer Required
- [ ] No explainer required — greenfield, no users

### 7. Cross-References and Subsystem Impact
- [ ] No consumer adapters (MAP, Chassis) write to `timeline_clips` yet — no adapter changes needed
- [ ] When render/export pipeline is built, it must use `get_keyframe_value` for all six properties

### 8. Verification
- [ ] All checklist items above pass
- [ ] `timeline_clips` has no `audio_level` column
- [ ] `clip_keyframes` table exists with correct schema, indexes, and FK

---

## Implementation Plan

**Execution environment:** VPS (shared PostgreSQL instance) + local application code

**Execution approach:**
1. In a single transaction: DROP COLUMN `audio_level` (if exists) + CREATE TABLE `clip_keyframes` + CREATE INDEXes
2. Update ORM models
3. Add `get_keyframe_value` helper
4. Update clip CRUD endpoints

**Estimated effort:** S-tier (schema simple; application helper is ~20 lines)

**Risk level:** Low — greenfield schema, no existing data

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
- **Extends:** DEC-CUE-2026-04-28-052000 (timeline data model — this decision supersedes the `audio_level` column from that doc)

---

**End of Decision Record**
