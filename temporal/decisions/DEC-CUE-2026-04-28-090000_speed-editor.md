# Decision: Speed Editor Integration via WebHID

**Created:** 2026-04-28
**Decision Status:** Approved
**Decision ID:** DEC-CUE-2026-04-28-090000
**Workspace:** cue
**Branch:** Operations

---

## Problem

The DaVinci Resolve Speed Editor is a hardware editing controller with a precision jog wheel and dedicated cut/trim buttons. Supporting it in Cue would allow frame-accurate clip positioning and fast edit operations without touching the mouse. The question is which integration path to take, given the Speed Editor uses a proprietary protocol.

**Background:**
The Speed Editor connects to the host machine via USB (HID) or Bluetooth (BLE). DaVinci Resolve activates the device with a proprietary authentication handshake; without it, the device operates in a degraded mode where the jog wheel sends standard HID wheel events and buttons send standard keyboard HID codes. The handshake protocol has been fully reverse-engineered and published.

**Current state:**
No hardware controller support exists. Cue is a web application (Python/Starlette backend, vanilla JS frontend) — the integration must work from the browser.

---

## Constitutional Alignment

- [x] **Values** — Creative output; makes the edit faster and more precise
- [x] **Temporal** — Cue active development Q2 2026
- [x] **Contexts** — Creator context: professional editing speed with hardware familiar from the Resolve workflow
- [x] **Modes** — Builder → Publisher: appropriate; this is a productivity enhancement, not a feature creep
- [x] **Algorithms** — Watched for Infinite Refinement: Phase 1 (standard HID, no auth) ships a working device without the full protocol implementation. Auth is Phase 2.

---

## Decision

Use the **WebHID API** for USB connection and **Web Bluetooth API** for BLE. Implement in two phases:

- **Phase 1 — standard HID (no auth):** Jog wheel as wheel events, buttons as keyboard HID codes. Works immediately, no reverse-engineered protocol required. Ships with the Edit screen.
- **Phase 2 — authenticated HID:** Implement the Blackmagic auth handshake to unlock precise jog data, jog mode switching, and LED control. Implement when Phase 1 proves insufficient for frame-accurate work.

The device is a **progressive enhancement** — Cue is fully usable without it. When not connected or on an unsupported browser (Firefox, Safari), the keyboard and mouse cover all the same operations.

---

## Platform constraints

| Feature | Chrome / Edge | Firefox | Safari |
|---------|--------------|---------|--------|
| WebHID (USB) | ✓ Supported | ✗ No | ✗ No |
| Web Bluetooth (BLE) | ✓ Supported | ✗ No | ✗ No (macOS via flag only) |

**Cue officially supports Chrome and Edge for Speed Editor functionality.** No fallback required — the editing UI works without the device. Document the browser requirement in the Edit screen UI (show a "Connect Speed Editor" button only on supported browsers; hide it on others).

---

## Phase 1 — Standard HID

No authentication. The device exposes:
- **Jog wheel** — standard HID wheel/scroll events
- **Buttons** — standard HID keyboard codes (most buttons send recognisable keycodes)

### Phase 1 connection flow

```javascript
const device = await navigator.hid.requestDevice({
    filters: [{ vendorId: 0x1EDB }]  // Blackmagic vendor ID
});
await device.open();
device.addEventListener('inputreport', handleHIDReport);
```

User must click a "Connect Speed Editor" button to trigger the permission prompt — WebHID cannot auto-connect without a user gesture.

### Phase 1 jog wheel behaviour

In standard HID mode, the jog wheel sends `wheel` delta values. Map them to timeline scrubbing:

```
timeline_offset_ms += wheel_delta * ms_per_tick
```

`ms_per_tick` is a user-configurable sensitivity setting (default: 1 frame at current zoom). Fine mode (holding a modifier key or pressing the JOG button on the device): `ms_per_tick = 1 frame`. Coarse mode: `ms_per_tick = 10 frames`.

When a clip is selected and the edit screen is in trim mode: the jog wheel trims the selected edit point instead of moving the playhead.

---

## Phase 2 — Authenticated HID

Implements the Blackmagic proprietary handshake to enable:
- Absolute jog wheel position (not just relative deltas) — enables faster, more precise scrubbing
- Jog mode switching (Jog / Scroll / Shuttle) via the on-device mode buttons
- LED state control (illuminate the active mode button)

### Auth protocol

The authentication uses a challenge-response sequence. Reference implementation:
- Protocol documented at: https://github.com/smunaut/blackmagic-misc (community reverse-engineering)
- The device sends a 6-byte challenge; the host computes a response using a published algorithm and sends it back
- Once authenticated, the device sends structured HID reports with absolute jog position and mode state

### Phase 2 jog modes

| Mode | Jog wheel behaviour | Cue mapping |
|------|-------------------|-------------|
| **Jog** | Fine, frame-by-frame | ±1 frame per tick — precise trim and position |
| **Scroll** | Coarser, position-based | Scroll through the timeline; speed proportional to turn rate |
| **Shuttle** | Variable-speed play | Play forward/backward at speed proportional to wheel angle; release to stop |

---

## Button mapping

Map Speed Editor buttons to Cue Edit screen operations. Unmapped buttons are ignored (no error).

### Transport

| Button | Cue action |
|--------|-----------|
| STOP | Stop playback; return playhead to in-point |
| PLAY (→) | Play from playhead |
| REVERSE (←) | Play backward from playhead (if supported by HLS player) |
| FAST FORWARD (↠) | Jump to next clip boundary |
| REWIND (↞) | Jump to previous clip boundary |

### Edit operations

| Button | Cue action |
|--------|-----------|
| SPLIT | Split clip at playhead position |
| UNDO | Undo last operation |
| REDO | Redo |
| IN | Set clip in-point to playhead position |
| OUT | Set clip out-point to playhead position |

### Mode switching (jog wheel context)

| Button | Cue effect on jog wheel |
|--------|------------------------|
| JOG (on wheel) | Phase 2: switch to frame-by-frame jog mode |
| SCROLL (on wheel) | Phase 2: switch to scroll mode |
| SHUTTLE (on wheel) | Phase 2: switch to shuttle mode |

### Unmapped (intentionally ignored for now)

RIPL DEL, CLOSE UP, PLACE ON TOP, SRC AUD, FULL VIEW, SYNC BIN, DESELECT, MARK CLIP, SOURCE OVERWRITE, APPEND, SMTH CUT, CLIP, HEADS, TAILS, PREV, NEXT, track selectors (V1, A1, etc.), numeric row.

These can be mapped to Cue operations as the Edit screen gains functionality. Document additions here.

---

## HID report parsing (Phase 1)

```javascript
function handleHIDReport(event) {
    const { reportId, data } = event;
    // Report structure varies by device mode.
    // In standard HID mode, buttons arrive as keycode reports (reportId 1).
    // Jog wheel arrives as wheel delta reports (reportId 2 or as standard input).
    // Log all unknown reports during development to build the map.
    console.log(`reportId=${reportId}`, new Uint8Array(data.buffer));
}
```

During Phase 1 implementation: log all incoming reports, press each button, turn the wheel, and record the reportId and data pattern. Build the mapping from observed behaviour, not assumption.

---

## Bluetooth (BLE) — deferred

Web Bluetooth is viable but adds complexity: the pairing flow is different, the device must be in BLE mode (hold a button on power-on), and battery drain is higher. The USB path covers the primary use case (desktop editing at a desk). BLE support is deferred to a future decision.

---

## Consequences

### Positive
- Jog wheel enables frame-accurate scrubbing that mouse dragging cannot match
- SPLIT, IN, OUT buttons are natural hardware equivalents of the most common Edit screen operations
- WebHID requires zero backend changes — purely a frontend JS module
- Progressive enhancement means it adds value without breaking anything

### Neutral
- Chrome/Edge only for the hardware feature; this matches the existing HLS/WebGL requirements anyway
- User must click a connect button once per browser session (WebHID permission model)

### Risks
- The Blackmagic auth protocol is reverse-engineered, not officially documented — it could change in a firmware update. Phase 1 standard HID is unaffected by this.
- If the user's Speed Editor is paired to Resolve via Bluetooth, it may not be available for USB HID at the same time. Document: connect via USB for Cue.

---

## Software Delivery

### Schema Impact
No schema changes.

### Data Integrity
No data impact.

### Affected Modules
- Edit screen frontend JS — add `speed-editor.js` module
- No backend changes required

### Test Coverage
- Unit test button → action mapping with a mocked HID device
- Integration smoke test: connect device, turn jog wheel, verify playhead moves
- Verify "Connect Speed Editor" button is hidden on Firefox/Safari

### Rollback Posture
A purely additive JS module — delete the file and remove the connect button to roll back. No side effects.

### Environment Notes
Chrome/Edge only. No flags required — WebHID is enabled by default in Chrome 89+.

---

## Execution Checklist

### 1. Schema / Migration
- [ ] No schema changes

### 2. Code Changes
- [ ] Create `static/js/speed-editor.js` module
- [ ] Detect WebHID support on page load; show/hide "Connect Speed Editor" button accordingly
- [ ] Implement `requestDevice` with Blackmagic vendor ID filter (`0x1EDB`)
- [ ] Log all incoming HID reports during dev to build the Phase 1 button map
- [ ] Implement jog wheel → playhead scrub (Phase 1: standard wheel delta)
- [ ] Implement button → Cue action mapping (STOP, PLAY, SPLIT, UNDO, REDO, IN, OUT)
- [ ] In trim mode: redirect jog wheel to selected edit point instead of playhead
- [ ] Phase 2 (separate task): implement Blackmagic auth handshake; switch to structured jog reports; add LED control

### 3. Documentation
- [ ] Update this doc with observed HID report map once Phase 1 is implemented (fill in the table under "HID report parsing")
- [ ] Update `_context/CHARTER.md` Architecture section to note Speed Editor as a supported input device

### 4. AI Usage Guide Required
- [ ] No AI guide required

### 5. Human Usage Guide Required
- [ ] Yes — short guide: how to connect the device, supported browsers, what each button does in Cue
  Create as `_guides/speed-editor.md` when the Edit screen ships

### 6. Change Explainer Required
- [ ] No explainer required — additive feature, no users yet

### 7. Cross-References and Subsystem Impact
- [ ] `_context/EDIT_SCREEN_INTERACTION.md` — Speed Editor drives the same operations described there; cross-reference already added

### 8. Verification
- [ ] Jog wheel moves playhead by 1 frame per tick in Jog mode
- [ ] SPLIT button splits clip at playhead
- [ ] Device not connected → no errors, all mouse/keyboard operations unaffected
- [ ] Firefox/Safari → "Connect Speed Editor" button not shown

---

## Implementation Plan

**Execution environment:** Local (frontend JS only)

**Execution approach:**
1. Add `speed-editor.js` with device detection and connection
2. Log all HID reports from a real device to build the button/wheel map
3. Implement jog → playhead and button → action dispatch
4. Wire into the Edit screen's existing action handlers

**Estimated effort:** M-tier (Phase 1); S-tier for each subsequent Phase 2 piece

**Risk level:** Low for Phase 1 (standard HID, no auth); Medium for Phase 2 (reverse-engineered protocol)

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
- **Interaction model:** `_context/EDIT_SCREEN_INTERACTION.md` — defines the operations this hardware drives

---

**End of Decision Record**
