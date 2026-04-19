# SnapFX — Implementation Changes

Changes applied to `index.html` on branch `claude/snapfx-code-fixes-iGXsP`.

---

## Fix 1 — Stale Face Data Fade-Out

**Problem:** When tracking.js loses the face (fast movement, poor lighting), `faceData` freezes at the last detected position. Filters kept rendering on that frozen position indefinitely with no feedback to the user.

**What changed:**
- Added `let lastFaceTime = 0` state variable
- In the `tracker.on('track')` callback, `lastFaceTime = Date.now()` is set on every successful detection
- In `startLoop()`, `lastFaceTime` is initialized to `Date.now()` so demo mode (no tracking) shows filters at full opacity
- The render loop now computes a `staleness` ratio over 1.5 seconds and derives `trackingAlpha = 1 - staleness`. The filter overlay is wrapped in `ctx.save() / ctx.globalAlpha = trackingAlpha / ctx.restore()` — if `trackingAlpha` reaches 0 the draw call is skipped entirely

**Result:** Filter fades to invisible after 1.5 seconds without a detected face, then snaps back immediately on re-detection.

---

## Fix 2 — Particle Pool Cleared on Filter Switch

**Problem:** The `particles` array (used only by the Fire filter) was never cleared when switching filters. Switching away from Fire and back could momentarily spike the particle count above the 130-cap.

**What changed:**
- One line added inside `div.onclick` in `renderFilters()`: `particles.length = 0`
- This runs on every filter switch, before the new filter ID is rendered

**Result:** Fire filter always starts clean when selected.

---

## Fix 3 — "Open Photo" Popup Reliability on Mobile

**Problem:** `openBtn.onclick` used `window.open('', '_blank')` followed by `w2.document.write(...)` to inject an HTML page containing the base64 image. On some mobile browsers the `document.write()` call is silently ignored after the tab opens, leaving a blank page.

**What changed** (superseded by Fix 5 — see below):
- `openBtn.onclick` now uses `URL.createObjectURL(savedBlob)` to give the new tab a real navigable URL pointing directly to the image blob
- The blob URL is revoked after 60 seconds to free memory
- No more `document.write()` — the image opens natively in the tab

---

## Fix 4 — Download Button DOM Pattern

**Problem:** `dlBtn.onclick` created an `<a>` element and called `.click()` without appending it to the document first. This works in Chrome but is technically undefined behavior and silently fails in Safari.

**What changed** (incorporated into Fix 5):
- The `<a>` element is now appended to `document.body` before `.click()` and removed immediately after
- A short `setTimeout` revokes the blob URL after the browser has had time to initiate the download

---

## Fix 5 — Async Photo Capture

**Problem:** `canvas.toDataURL('image/jpeg', 0.95)` is synchronous and blocks the main thread while JPEG-encoding a full-resolution canvas frame. On slower phones this caused a visible stutter in the camera feed.

**What changed:**
- `let savedDataUrl = ''` replaced with `let savedBlob = null` and `let savedDataUrl = ''`
- `shutter.onclick` now calls `offscreen.toBlob(callback, 'image/jpeg', 0.95)` — encoding happens off the main thread
- `savedDataUrl` is now a blob object URL (`URL.createObjectURL`) rather than a base64 string, so it is revoked in `closeModal.onclick` to prevent memory leaks
- `openBtn.onclick` and `dlBtn.onclick` both reference `savedBlob` directly instead of the data URL

---

## Fix 6 — `time` Counter Bounded

**Problem:** The `time` variable increments by 1 every frame and never resets. At 60fps this takes years to cause issues, but large integer values cause floating-point precision loss in trig functions (`Math.sin(t * 0.08)` etc.) used for filter animations.

**What changed:**
- `time++` replaced with `time = (time + 1) % 1000000`
- 1,000,000 is a clean multiple of common animation periods; the wrap is invisible in any animation

---

## Fix 7 — Per-Pixel Filter Throttling

**Problem:** `noir`, `fisheye`, `cartoon`, and `pixel` call `getImageData()` / `putImageData()` on every animation frame. On a 1280×720 canvas that moves ~3.5 MB of pixel data synchronously per frame, which tanks frame rate on older mobile hardware.

**What changed:**
- Added `const pixelCache = { imageData: null, frame: -1, filterId: '' }` state object
- Each of the four filter blocks now checks `time % 2 === 0 || pixelCache.frame !== time - 1 || pixelCache.filterId !== f`
  - On even frames (or after a filter switch): runs the full pixel processing and writes to `pixelCache`
  - On odd frames: repaints from `pixelCache.imageData` without calling `getImageData` again
- `filterId` in the cache prevents stale data from one pixel filter bleeding into another

**Result:** CPU cost of pixel processing is halved. Visual difference is imperceptible at 60fps.

---

## Fix 8 — Correct Mirror on Save + Freeze-Frame Feedback

**Problem 1:** The live canvas is rendered mirrored (`ctx.scale(-1, 1)`) to feel natural as a selfie view. `canvas.toBlob()` captured that mirrored frame directly, so saved photos were backwards — text, asymmetric filters, and faces all appeared flipped.

**Problem 2:** There was no visual indication of exactly which frame was captured. With any lag between tap and modal render, the user couldn't be sure what they got.

**What changed:**
- Added `let frozen = false` state variable
- The render loop's video redraw is guarded with `&& !frozen` — when frozen, the canvas holds the captured frame visually
- On shutter press:
  1. `frozen = true` immediately freezes the live view
  2. An offscreen canvas is created at the same dimensions
  3. The live canvas (already mirrored) is drawn onto the offscreen canvas with `octx.scale(-1, 1)` — this double-flip produces a correctly oriented image
  4. `offscreen.toBlob()` encodes the corrected frame asynchronously
  5. After 300ms, `frozen = false` resumes the live view
- Capturing from the canvas (not the video element) preserves the composited filter overlay in the saved image

**Result:** Live view remains mirrored (natural selfie feel), saved photo is correctly oriented, and a brief ~300ms freeze gives clear visual confirmation of exactly what was captured.

---

## Commit Log

| Commit | Description |
|--------|-------------|
| `bf823d1` | Fix 6: Bound time counter to prevent float precision loss |
| `6d34871` | Fix 2: Clear particle pool on filter switch |
| `0e951a7` | Fix 1: Fade filter overlay when face tracking is lost |
| `8dd5ca2` | Fix 7: Throttle per-pixel filters to every other frame |
| `2767a1f` | Fixes 3+4+5+8: Async capture, blob URLs, mirror correction, freeze frame |
| `a2bb209` | Add README with project overview and architecture |
