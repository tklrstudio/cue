# Decision: Render Pipeline

**Created:** 2026-04-28
**Decision Status:** Approved
**Decision ID:** DEC-CUE-2026-04-28-100000
**Workspace:** cue
**Branch:** Operations

---

## Problem

Cue needs to export a timeline as a finished media file. The render must composite multi-track video with keyframe transforms, mix audio across tracks, and produce an output file — without Cue permanently storing source media or requiring re-upload of assets already in B2.

**Background:**
Assets in `cue.assets` are already encoded and stored in B2 as HLS manifests + segments. The CHARTER architectural invariant states: "Export renders pull directly from B2 via source HLS — no re-encoding of already-encoded assets unless required."

**Current state:**
No render pipeline exists. This decision establishes the architecture before implementation begins.

---

## Constitutional Alignment

- [x] **Values** — Creative output; the render is the product of every edit decision
- [x] **Temporal** — Cue active development Q2 2026
- [x] **Foundations** — Resources: stream-copy path avoids unnecessary re-encode cost and time
- [x] **Modes** — Builder → Publisher: render is the handoff point from Cue to the publishing layer

---

## Decision

ffmpeg-based render running on the VPS as an async job. Two execution paths selected per clip based on whether non-identity keyframes exist:

- **Stream copy path** — clips with identity keyframes (no transforms, audio level 1.0): ffmpeg copies segments directly from B2 without re-encoding. Fast, zero quality loss.
- **Re-encode path** — clips with non-identity keyframes: ffmpeg builds a filter graph that applies transforms and audio gain, then encodes.

Output: H.264 MP4 to a staging location in B2, then a webhook to the requesting system on completion.

---

## Input — pulling from B2

Render does **not** download assets to disk. ffmpeg reads HLS segments directly from B2 via signed URLs, using the `-allowed_extensions ALL` flag for m3u8 inputs.

```
ffmpeg -allowed_extensions ALL -i "https://b2.tklr.net/<bucket>/<path>.m3u8" ...
```

The `.m3u8` manifest URL is `cue.assets.hls_manifest_url`. Segment URLs are resolved from the manifest at render time.

For trim (in/out points): ffmpeg `-ss` and `-to` flags applied to each input. Millisecond precision: `-ss 00:00:01.240 -to 00:00:04.780`.

---

## The two render paths

### Path A — stream copy (no keyframes)

A clip qualifies for stream copy if:
- No `clip_keyframes` rows exist for it, OR
- All existing keyframe rows have identity values (position 0/0, scale 1.0/1.0, rotation 0, audio_level 1.0)

```bash
ffmpeg \
  -allowed_extensions ALL \
  -ss <in_point_s> -to <out_point_s> \
  -i "<hls_manifest_url>" \
  -c copy \
  <output_segment.ts>
```

Each clip produces a `.ts` segment. Segments are concatenated via the ffmpeg concat demuxer into the final output.

**Constraint:** All stream-copied clips must share compatible codec parameters (same resolution, codec, frame rate). If they differ, the mismatching clips fall back to Path B.

### Path B — re-encode with filter graph

Used when a clip has non-identity keyframes, or when stream copy is incompatible.

The filter graph reads keyframes for the clip and applies them as ffmpeg filters. Since animation is not yet supported (all keyframes are `interpolation='hold'`), the value at `time_ms=0` is used for the entire clip duration — a single static filter.

```python
# Build per-clip filter fragment
def clip_filter(clip, keyframes):
    kf = {k['property']: k['value'] for k in keyframes}
    px   = kf.get('position_x', 0)
    py   = kf.get('position_y', 0)
    sx   = kf.get('scale_x',   1.0)
    sy   = kf.get('scale_y',   1.0)
    rot  = kf.get('rotation',  0.0)
    gain = kf.get('audio_level', 1.0)

    video_filter = f"scale=iw*{sx}:ih*{sy},rotate={rot}*PI/180,pad=W:H:(W-iw)/2+{px}:(H-ih)/2+{py}"
    audio_filter = f"volume={gain}"
    return video_filter, audio_filter
```

When animation lands, this function will interpolate `value` across keyframe `time_ms` values and generate a time-varying filter expression.

---

## Multi-track compositing

The render pipeline builds the output canvas (`timelines.resolution_w × timelines.resolution_h`) by layering video tracks bottom-up (V1 at the bottom, V2 above, etc.) using ffmpeg `overlay` filters.

For each timeline position `t`:
1. Identify all clips active at `t` (where `timeline_position_ms <= t < clip_end_on_timeline_ms`)
2. Sort by track index ascending (V1 first)
3. Overlay in order

Audio tracks are mixed with `amix` or `amerge`, with per-clip gain from keyframes applied before mixing.

---

## Output format

| Property | Value |
|----------|-------|
| Container | MP4 |
| Video codec | H.264 (libx264), CRF 18 (visually lossless) |
| Audio codec | AAC, 320 kbps |
| Frame rate | From `timelines.frame_rate` |
| Resolution | From `timelines.resolution_w × resolution_h` |
| Destination | B2 at `cue/renders/<timeline_id>/<render_id>.mp4` |

**Future:** ProRes output option for clips going into further post-production. Not in scope until a consumer requests it.

---

## Job model

Render is an async VPS job — not a web request. Flow:

1. Client calls `POST /api/timelines/{id}/render`
2. Cue API inserts a render job (new table: `cue.render_jobs`, see Schema Impact)
3. VPS worker picks up the job, runs ffmpeg, writes to B2
4. On completion: updates `render_jobs.status` to `complete` and fires a webhook to `render_jobs.callback_url`
5. Client polls `GET /api/renders/{render_id}` or waits for the webhook

Progress: the VPS worker parses ffmpeg's `progress` output and writes `render_jobs.progress_pct` (0–100) on each ffmpeg progress tick. The client can poll this for a progress bar.

---

## Schema Impact

New table required:

```sql
CREATE TABLE cue.render_jobs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    timeline_id     UUID NOT NULL REFERENCES cue.timelines(id),
    status          TEXT NOT NULL DEFAULT 'queued',
    -- 'queued' | 'running' | 'complete' | 'failed'
    progress_pct    INT,
    output_b2_key   TEXT,
    callback_url    TEXT,
    error           TEXT,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

This table was not in DEC-CUE-2026-04-28-052000. Add it alongside the original six tables.

---

## Consequences

### Positive
- Stream copy path for simple cuts is near-instant and produces no quality loss
- Filter graph is built programmatically — adding keyframe animation later is isolated to the filter-building function
- ffmpeg on VPS keeps the web server stateless
- B2-direct input eliminates local storage for source assets

### Neutral
- Re-encode path is slower (minutes for a long timeline). Expected and unavoidable when transforms are applied.
- All clips in a stream-copy render must have compatible codecs. Incompatible clips silently fall back to re-encode.

### Risks
- Signed B2 URL expiry during a long render. Mitigation: generate long-lived signed URLs (24hr) at job start, not at ffmpeg invocation time.
- ffmpeg filter graph complexity grows with track count and clip count. For timelines with many overlapping transforms, the filter graph string can become very long. Not a practical concern for initial use cases (TKLR Shorts are short).
- HLS segment boundary alignment: trim points may not align to HLS segment boundaries, causing ffmpeg to decode partial segments. This is handled automatically by ffmpeg but adds a small overhead.

---

## Software Delivery

### Schema Impact
- New table: `cue.render_jobs` (DDL above)
- Backwards compatible: yes (additive)

### Data Integrity
No existing rows affected.

### Affected Modules
- New: `render/pipeline.py` — ffmpeg orchestration, filter graph builder, B2 signed URL generation
- New: `render/worker.py` — VPS worker that polls `render_jobs` and dispatches to pipeline
- New: API endpoint `POST /api/timelines/{id}/render`
- New: API endpoint `GET /api/renders/{render_id}`

### Test Coverage
- Unit test filter graph builder: identity keyframes → stream copy path selected; non-identity keyframes → correct filter string generated
- Integration test: render a 3-clip timeline with one transformed clip; verify output file exists in B2 and plays correctly
- Test B2 signed URL expiry handling
- Test `render_jobs.progress_pct` updates during a running render

### Rollback Posture
New table + new module — drop table and delete module to roll back. No impact on existing data.

### Environment Notes
ffmpeg must be installed on the VPS. Version ≥ 5.0 required for full HLS input support. Verify with `ffmpeg -version` before deploying.

---

## Execution Checklist

### 1. Schema / Migration
- [ ] Add `cue.render_jobs` table (DDL above)
- [ ] Verify `ffmpeg --version` ≥ 5.0 on VPS

### 2. Code Changes
- [ ] `render/pipeline.py` — build filter graph from `clip_keyframes`; select stream copy vs re-encode per clip
- [ ] `render/pipeline.py` — generate 24hr signed B2 URLs for all input manifests at job start
- [ ] `render/pipeline.py` — parse ffmpeg progress output and write to `render_jobs.progress_pct`
- [ ] `render/worker.py` — poll `render_jobs` for `status='queued'` and dispatch
- [ ] `render/worker.py` — fire `callback_url` webhook on completion/failure
- [ ] `POST /api/timelines/{id}/render` endpoint
- [ ] `GET /api/renders/{render_id}` endpoint

### 3. Documentation
- [ ] Update `_context/CHARTER.md` Architecture section to include `cue.render_jobs`

### 4. AI Usage Guide Required
- [ ] No AI guide required

### 5. Human Usage Guide Required
- [ ] No human guide required at this stage

### 6. Change Explainer Required
- [ ] No explainer required — greenfield

### 7. Cross-References and Subsystem Impact
- [ ] Chassis Publisher will call the render API to trigger export — ensure the webhook payload is documented before Chassis integration begins
- [ ] MAP has no render requirement currently

### 8. Verification
- [ ] `render_jobs` table created
- [ ] End-to-end render of a test timeline produces a valid MP4 in B2
- [ ] Progress polling works during a running render
- [ ] Webhook fires on completion

---

## Implementation Plan

**Execution environment:** VPS (ffmpeg) + Cue API

**Execution approach:**
1. Create `render_jobs` table
2. Build `pipeline.py` (filter graph + ffmpeg invocation)
3. Build `worker.py` (job polling + dispatch + webhook)
4. Wire API endpoints
5. Integration test against a real timeline

**Estimated effort:** M-tier (4–6 tasks)

**Risk level:** Medium — ffmpeg filter graph complexity and B2 URL handling are the main unknowns

---

## Problems Solved

No significant problems at design time.

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
- **Depends on:** DEC-CUE-2026-04-28-052000 (timeline data model — source of clip and timeline data)
- **Depends on:** DEC-CUE-2026-04-28-080000 (clip keyframes — source of transform and audio level values)

---

**End of Decision Record**
