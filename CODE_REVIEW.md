# Code Review: `index.html`

Date: 2026-04-07

## Scope
- Reviewed the single-page app implementation in `index.html` (layout/CSS/JS behavior, keyboard handling, modal interaction, and reset/animation flows).

## Findings

### 1) Modal does not block keyboard input to the editor (High)
**Where**
- `createPaperArea()` focuses `paperDiv` during startup.
- The info modal is shown on `DOMContentLoaded`, but the app only blocks page scrolling (`body.modal-open`) and does not disable key handling while modal is visible.

**Impact**
- Users can accidentally type into the page while reading the modal because `paperDiv` can still receive `keydown` events.
- This conflicts with the modal's intent as a “read before you start” gate.

**Suggested fix**
- Add a guard at the top of `handleKey()`:
  - If modal is visible, `e.preventDefault(); return;`
- Or move focus to the modal confirm button until dismissal.

---

### 2) Asynchronous key actions can outlive reset transitions (Medium)
**Where**
- Several input paths use `setTimeout` (`SHORT_DELAY`, `ENTER_ACTION_DELAY`) and then mutate shared state (`currentData`, `cursorRow`, `cursorCol`).
- `hardReset()` can recreate the paper and state while delayed callbacks from prior input are still pending.

**Impact**
- Possible state races: delayed callbacks may apply edits to a newly reset page, causing unexpected characters/cursor movement right after reset.

**Suggested fix**
- Track pending timer IDs in a set and clear them in `hardReset()` and before creating a new paper.
- Alternatively use a monotonically increasing session token and ignore delayed callbacks from stale sessions.

---

### 3) Event handlers are bound per-paper without explicit teardown strategy (Low)
**Where**
- `createPaperArea()` binds `keydown`, `keyup`, and `mousedown` handlers each time a new paper is created.

**Impact**
- Current implementation removes old `paperDiv`, so this is mostly safe.
- Still, explicit teardown patterns reduce risk if lifecycle changes later.

**Suggested fix**
- Consider centralizing keyboard handling at document level with active-instance guards, or keep references and remove listeners before element removal.

## Positives
- Good separation of concerns with named sections (settings, state, layout, rendering, PDF, reset).
- Preview/PDF flows reuse normalized page records (`normalizePageRecord`) consistently.
- Defensive handling around audio playback promises is robust for browser autoplay restrictions.

## Recommended next steps
1. Fix modal keyboard gating first (user-visible correctness).
2. Add timer/session cancellation to eliminate reset races.
3. Add a small regression checklist for:
   - typing while modal open,
   - rapid key input + hard reset,
   - enter animation while toggling side panel.
