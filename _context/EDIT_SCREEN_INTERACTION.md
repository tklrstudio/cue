# Edit Screen — Timeline Interaction Design

**Purpose:** Canonical reference for timeline editing interactions — precision movement, trimming, snapping, and keyboard/hardware control
**Status:** Canonical
**Scope:** Edit screen UI
**Created:** 2026-04-28
**Last Updated:** 2026-04-28

---

## Schema reference (columns these operations touch)

```
cue.timeline_clips
  timeline_position_ms   BIGINT   — where on the timeline this clip starts
  in_point_ms            BIGINT   — source start point within the asset
  out_point_ms           BIGINT   — source end point within the asset

cue.timelines
  frame_rate             NUMERIC  — used for timecode display and frame-snapping
```

**Clip duration on the timeline** (not a stored column — derive it):
```
duration_on_timeline_ms = out_point_ms - in_point_ms
clip_end_on_timeline_ms = timeline_position_ms + duration_on_timeline_ms
```

---

## The three edit operations

### 1. Move (drag clip body)

Shifts the clip along the timeline. In/out points do not change; only the position does.

```
timeline_position_ms += delta_ms
```

The clip's audio or video pair (same `placement_group_id`) moves by the same delta. Both rows always move together.

**Constraint:** `timeline_position_ms >= 0`. Cannot drag before the start of the timeline.

---

### 2. Trim head (drag left edge)

Adjusts where the clip starts within the source asset AND shifts the clip's timeline position by the same delta, so the clip's tail stays fixed.

```
in_point_ms            += delta_ms
timeline_position_ms   += delta_ms
```

Both change by the same amount — the clip shrinks or grows from the left while its right edge stays put.

**Constraints:**
- `in_point_ms >= 0` — cannot go before the start of the source asset
- `in_point_ms < out_point_ms` — clip must have non-zero duration
- `timeline_position_ms >= 0`

---

### 3. Trim tail (drag right edge)

Adjusts where the clip ends within the source asset. Timeline position does not move; the right edge of the clip on the timeline moves.

```
out_point_ms += delta_ms
```

**Constraints:**
- `out_point_ms > in_point_ms` — clip must have non-zero duration
- `out_point_ms <= asset.duration_ms` — cannot go past the end of the source media (null for images — unlimited hold duration)

---

## Gap behaviour (non-ripple)

Cue uses a **non-ripple** edit model: moving or trimming one clip never automatically moves others. Gaps are allowed and normal. When a clip is moved inward (leaving a gap behind), the gap is just empty space.

Ripple mode (auto-close gaps, auto-shift downstream clips) is out of scope until explicitly decided. Do not implement it speculatively.

---

## Snap

### Snap targets

When dragging a clip or trim handle, snap to any of the following if within the snap threshold:

| Target | Value |
|--------|-------|
| Start of any other clip | `other_clip.timeline_position_ms` |
| End of any other clip | `other_clip.timeline_position_ms + (other_clip.out_point_ms - other_clip.in_point_ms)` |
| Timeline start | `0` |
| Playhead position | current playhead `ms` |

Snap applies **across all tracks** — a clip on V2 snaps to the boundary of a clip on A1. This is the default NLE convention and is almost always what the editor wants.

### Snap threshold

Snap activates when the drag handle is within `snap_threshold_px` pixels of a target, converted to ms at the current zoom level:

```
snap_threshold_ms = snap_threshold_px / px_per_ms
```

Typical value: `snap_threshold_px = 8`. Recalculate on every zoom change.

### Snap indicator

Draw a vertical snap line at the target position when snapping is active. Remove it on release or when the drag moves outside the threshold.

### Snap override

Hold `Alt` while dragging to disable snap temporarily. This is the universal NLE convention.

---

## Precision timecode input

Every clip position and every trim handle should accept a typed timecode value.

### Format

```
HH:MM:SS:FF
```

`FF` = zero-based frame number, `0` to `ceil(frame_rate) - 1`.

Display examples at 25 fps: `00:00:10:12` = 10 seconds and 12 frames.

### Timecode ↔ milliseconds conversion

```python
# timecode → ms
def timecode_to_ms(h, m, s, f, frame_rate):
    total_frames = (h * 3600 + m * 60 + s) * frame_rate + f
    return round(total_frames * 1000 / frame_rate)

# ms → timecode
def ms_to_timecode(ms, frame_rate):
    total_frames = round(ms * frame_rate / 1000)
    f = total_frames % frame_rate
    total_s = total_frames // frame_rate
    s = total_s % 60
    m = (total_s // 60) % 60
    h = total_s // 3600
    return f"{h:02}:{m:02}:{s:02}:{int(f):02}"
```

### Where it appears

- **Clip position field** — shows/edits `timeline_position_ms`
- **In point field** — shows/edits `in_point_ms`
- **Out point field** — shows/edits `out_point_ms`
- **Duration field** — read-only display of `out_point_ms - in_point_ms`; editing it adjusts `out_point_ms`
- **Playhead position field** — in the transport bar; typing here seeks the playhead

---

## Keyboard nudge

When a clip is selected, nudge it without dragging:

| Key | Delta |
|-----|-------|
| `←` / `→` | ±1 frame (`round(1000 / frame_rate)` ms) |
| `Shift + ←` / `Shift + →` | ±10 frames |
| `Shift + Alt + ←` / `Shift + Alt + →` | ±1 second (1000 ms) |

When a trim handle is selected (keyboard focus on a handle), the same keys trim rather than move.

---

## Zoom

Timeline zoom controls how many ms fit per pixel (`ms_per_px`). Zoom level affects:
- Snap threshold calculation (see above)
- How far a drag delta moves in ms: `delta_ms = delta_px * ms_per_px`
- Timecode display granularity (at tight zoom, show frames; at wide zoom, show only seconds)

Zoom is UI state only — no schema column needed.

---

## Multi-clip select and group move

When multiple clips are selected, dragging any one of them moves all selected clips by the same `delta_ms`. Each clip's `timeline_position_ms` updates independently. Trim operations apply only to the clip whose handle is being dragged, regardless of selection.

---

## Speed Editor integration

See `DEC-CUE-2026-04-28-090000_speed-editor.md` for the jog wheel and button mapping that drives these same operations via hardware.

---

**End Edit Screen Interaction Design**
