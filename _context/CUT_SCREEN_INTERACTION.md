# Cut Screen — Interaction Design

**Purpose:** Canonical reference for the Cut screen — asset library browsing, preview, and clip placement onto the timeline
**Status:** Canonical
**Scope:** Cut screen UI
**Created:** 2026-04-28
**Last Updated:** 2026-04-28

---

## Layout

Three panels, horizontal split:

```
┌─────────────┬──────────────────────────────┬──────────────────┐
│  Bins        │  Asset grid / list           │  Preview player  │
│  sidebar     │                              │                  │
│  (collapsible│  [clip cards]                │  [HLS player]    │
│   left)      │                              │  in/out controls │
│              │                              │  place controls  │
└─────────────┴──────────────────────────────┴──────────────────┘
```

- **Bins sidebar** — left, collapsible. Lists all bins; click to filter the asset grid.
- **Asset grid** — centre, primary interaction area. Shows filtered/searched assets as cards.
- **Preview player** — right. HLS player; activated by selecting an asset. Shows in/out markers and place controls.

The timeline is **not visible** on the Cut screen. Cut = select and mark; Edit = arrange and refine.

---

## Bins sidebar

- Lists all `cue.bins` rows in alphabetical order, with a count of assets in each
- **All Assets** is a virtual bin at the top (no filter applied)
- AI-generated bins shown with a subtle indicator (e.g. a small ✦ icon)
- Click a bin → filter the asset grid to that bin's assets
- Active bin is highlighted; click again to deselect (returns to All Assets)
- Bins cannot be created, renamed, or deleted from the Cut screen — that's a library management concern for a future screen

---

## Asset grid

### Card content

Each card shows:
- Thumbnail (first frame, or poster image from asset metadata)
- Duration (formatted `HH:MM:SS` or `MM:SS` depending on length)
- Asset type indicator (video / audio / image)
- Source system tag (MAP / Chassis / etc.) — subdued
- A brief title or identifier from `asset.metadata` if available

### View modes

Toggle between **grid** (default, thumbnail-forward) and **list** (denser, shows more metadata columns). State is per-session, not persisted.

### Search

A search field above the grid filters by `asset.metadata` content (full-text, client-side for reasonable library sizes; server-side search endpoint for large libraries). Search and bin filter apply together — search within the active bin.

### Selection

- Click card → select asset, load into preview player
- Selected card is highlighted
- Only one asset selected at a time on the Cut screen (multi-select is an Edit screen concept)

### Sorting

Sort controls above the grid: by ingested date (default, newest first), duration, source system. No drag-reorder — the grid is a library view, not an edit decision.

---

## Preview player

### Playback

- `Space` — play / pause
- `J` / `K` / `L` — rewind / pause / play (standard NLE transport)
- `←` / `→` — step one frame
- `Shift + ←` / `Shift + →` — step 10 frames
- Click on the scrub bar — seek to that position
- Jog wheel (Speed Editor) — scrub frame by frame when the player has focus

### In / out point marking

Mark the portion of the clip you want to use before placing it on the timeline.

| Key | Action |
|-----|--------|
| `I` | Set in-point to current playhead position |
| `O` | Set out-point to current playhead position |
| `Shift + I` | Clear in-point (reset to asset start) |
| `Shift + O` | Clear out-point (reset to asset end) |
| `X` | Select entire clip (reset both in and out) |

The marked region is shown as a highlighted bar on the scrub track. Duration of the marked region is displayed numerically.

In/out points here are **preview-only** — they pre-fill `in_point_ms` and `out_point_ms` when the clip is placed on the timeline but do not write to the database until placement.

### Timecode display

Shows current playhead position and marked duration in `HH:MM:SS:FF` format, using the source asset's native frame rate.

---

## Placing a clip on the timeline

### Target track

A track selector below the preview player: **V1, V2, A1, A2**, etc. Default: V1 (lowest available video track). For audio-only assets: default A1. This sets which track the clip lands on.

### Placement methods

| Method | Behaviour |
|--------|-----------|
| Drag card from grid to timeline | Drop position determines `timeline_position_ms`; snaps to clip boundaries |
| **Append** (`F` or toolbar button) | Places clip immediately after the last clip on the selected track. No gap. |
| **Insert at playhead** (`Shift + F`) | Places clip at the current timeline playhead position. Existing clips are not moved (non-ripple). |

Append is the primary flow for rough-cutting — tap `F` to send clip after clip to the timeline without touching the mouse.

### What gets written

On placement:
- One `timeline_clips` row for video (if asset has video), one for audio (if asset has audio)
- Both share a `placement_group_id`
- `in_point_ms` and `out_point_ms` populated from the preview player's marked region
- `timeline_position_ms` set to drop position or calculated append position
- No `clip_keyframes` rows created — identity defaults apply

---

## Keyboard reference (Cut screen)

| Key | Action |
|-----|--------|
| `Space` | Play / pause preview |
| `I` / `O` | Set in / out point |
| `Shift + I` / `Shift + O` | Clear in / out point |
| `X` | Select full clip |
| `J` / `K` / `L` | Rewind / stop / play |
| `←` / `→` | Step one frame |
| `Shift + ←` / `Shift + →` | Step 10 frames |
| `F` | Append to selected track |
| `Shift + F` | Insert at playhead position |
| `Cmd + F` | Focus search field |
| `Escape` | Clear search / deselect bin filter |

---

## Speed Editor on the Cut screen

| Control | Cut screen action |
|---------|-----------------|
| Jog wheel | Scrub preview player |
| PLAY / STOP | Transport |
| IN / OUT | Set in / out point |
| APPEND (mapped to `F`) | Append to selected track |

---

**End Cut Screen Interaction Design**
