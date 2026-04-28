# Edit Screen — Timeline Interaction Design

**Purpose:** Canonical reference for timeline editing interactions — precision movement, trimming, snapping, multi-select, zoom/pan, and keyboard/hardware control
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

## Zoom and pan

### The zoom variable

```
px_per_ms   — pixels per millisecond. The single source of truth for zoom level.
              Stored as UI state; never persisted to the database.
```

Everything zoom-dependent derives from it:

```
delta_ms            = delta_px / px_per_ms          // drag delta conversion
snap_threshold_ms   = snap_threshold_px / px_per_ms // snap window in time
clip_width_px       = duration_on_timeline_ms * px_per_ms
position_px         = timeline_position_ms * px_per_ms
```

### Min / max zoom

| Level | `px_per_ms` | 1 second = | 1 frame at 25fps = | Useful for |
|-------|-------------|------------|---------------------|------------|
| **Min** | `0.002` | 2 px | 0.08 px | Overview of a 60-min session (120 px) |
| **Default** | `0.04` | 40 px | 1.6 px | ~30 s visible in a 1200 px viewport |
| **Max** | `2.0` | 2000 px | 80 px | Frame-accurate trim and keyframe work |

Clamp `px_per_ms` to `[0.002, 2.0]` at all times.

### Zoom controls

| Input | Effect |
|-------|--------|
| `Cmd + scroll up` | Zoom in (increase `px_per_ms`) |
| `Cmd + scroll down` | Zoom out (decrease `px_per_ms`) |
| `=` / `+` | Zoom in one step |
| `-` | Zoom out one step |
| `Shift + Z` | Fit entire timeline in viewport (set `px_per_ms` so the last clip's end fits) |

Each scroll tick or key step multiplies / divides `px_per_ms` by `1.15`. Ten ticks doubles or halves the zoom — feels smooth and consistent regardless of starting level.

**`Cmd + scroll` note:** This conflicts with the browser's native page zoom. Call `event.preventDefault()` on wheel events when `Cmd` is held and the cursor is over the timeline canvas. Do not intercept `Cmd + scroll` outside the timeline area.

### Zoom origin (zoom-to-cursor)

Zoom always centres on the cursor's timeline position, not the viewport centre. The point under the cursor stays fixed on screen while everything else scales around it.

```javascript
// Before changing px_per_ms:
const cursor_ms = (cursor_x + scroll_left) / old_px_per_ms;

// After setting new px_per_ms:
scroll_left = cursor_ms * new_px_per_ms - cursor_x;
```

When zooming via keyboard (no cursor on timeline), zoom to the playhead position instead.

### Horizontal pan (scroll)

| Input | Effect |
|-------|--------|
| `Shift + scroll up/down` | Pan timeline left / right |
| Scrollbar drag | Pan |
| Trackpad two-finger horizontal swipe | Pan (native browser behaviour — no override needed) |

Pan step per scroll tick: `120 px` (fixed, independent of zoom level). This feels consistent — at any zoom level, one tick moves the same screen distance.

Do **not** use unmodified scroll for horizontal pan. Unmodified scroll is reserved for vertical scrolling when the track list is taller than the viewport.

### Timecode ruler granularity

Adjust the ruler tick interval based on zoom so labels never overlap. Suggested thresholds (tweak to taste):

| `px_per_ms` | Show ticks at |
|-------------|---------------|
| < 0.005 | Every 60 s |
| 0.005 – 0.02 | Every 10 s |
| 0.02 – 0.1 | Every 1 s |
| 0.1 – 0.5 | Every 100 ms (every ~2–3 frames at 25fps) |
| ≥ 0.5 | Every frame |

---

## Multi-clip selection

### Selection methods

| Method | Effect |
|--------|--------|
| Click clip | Select that clip, deselect all others |
| `Shift + click` clip | Add clip to selection (or remove if already selected) |
| Click empty timeline area | Deselect all |
| Drag on empty area | Box select — draw a rectangle; select all clips that overlap it |
| `Cmd + A` | Select all clips on all tracks |

### Box select

Click-drag starting from empty timeline space (not on a clip or handle). While dragging, draw a semi-transparent selection rectangle. On release, any clip whose rendered rectangle intersects the box is added to the selection. Does not deselect clips outside the box if `Shift` is held during the drag.

A clip "intersects" the box if any part of it falls within the time range AND the track row overlaps the vertical range of the box.

### Group move

When multiple clips are selected, dragging any one of them moves **all** selected clips by the same `delta_ms`. Each clip's `timeline_position_ms` updates independently. All `placement_group_id` pairs within the selection move together regardless of which clip is dragged.

**Constraint:** If moving the group would push any clip before `timeline_position_ms = 0`, clamp the whole group — do not move the group at all (or clamp the delta so the earliest clip lands at 0).

### Trim with multi-select

Trim operations (drag left or right edge) apply only to the clip whose handle is being dragged, regardless of what else is selected. Other selected clips do not trim. This is standard NLE behaviour.

### Keyboard nudge with multi-select

Nudge keys (`←` / `→` etc.) apply to all selected clips simultaneously — same delta for all.

---

## Speed Editor integration

See `DEC-CUE-2026-04-28-090000_speed-editor.md` for the jog wheel and button mapping that drives these same operations via hardware.

---

**End Edit Screen Interaction Design**
