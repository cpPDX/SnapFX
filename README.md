# SnapFX

A single-file webcam face-filter camera with 35+ live filters. No framework, no build step, no backend. Open in a browser and go.

**Live demo:** [cppdx.github.io/snapfx](https://cppdx.github.io/snapfx/)

---

## Features

- 35+ filters across Animals, Cute, Character, FX, and Photo categories
- Real-time face tracking via [tracking.js](https://trackingjs.com/) — falls back to demo mode if unavailable
- Adjustable filter intensity slider
- Photo capture with flash animation and modal preview
- Download photo or open in new tab (iPhone: long-press → Add to Photos)

## Usage

HTTPS is required for camera access. Serve over GitHub Pages or any HTTPS host — do not open from `file://`.

```
git clone https://github.com/cppdx/snapfx.git
cd snapfx
# Upload index.html to GitHub Pages or any HTTPS server
```

- **Allow camera** when prompted
- **Pick a filter** from the bottom tray
- **Adjust intensity** with the slider
- **Tap the shutter** to capture

## Project Structure

```
snapfx/
└── index.html   ← entire app (HTML + CSS + JS, ~720 lines)
```

Deploy by uploading `index.html` to GitHub Pages. Changes go live within ~60 seconds.

## Architecture

```
getUserMedia() → <video> → canvas.drawImage()
                         → tracking.js face detection
                         → drawFilter() overlay
                         → requestAnimationFrame loop
```

All filter drawing happens in `drawFilter()` — one large function with an `if` block per filter. Face position is normalized (0–1) and converted to pixels by the `B()` helper. The `time` counter drives animations (sin/cos waves, hue rotation).

Per-pixel filters (`noir`, `fisheye`, `cartoon`, `pixel`) use `getImageData`/`putImageData` and are throttled to every other frame to reduce CPU load on mobile.

## Known Limitations

- **No landmark detection** — tracking.js provides a bounding box only (no eye/nose/lip points). MediaPipe would be the right upgrade.
- **iOS camera requires Safari over HTTPS** — Chrome on iPhone won't work (Apple platform restriction).
- **Saving to iPhone Camera Roll** requires the long-press workaround — JavaScript cannot trigger native "Save to Photos."

## Browser Support

| Browser | Camera | Filters | Save |
|---------|--------|---------|------|
| Chrome (desktop/Android) | ✓ | ✓ | ✓ |
| Safari (iOS/macOS) | ✓ | ✓ | Long-press |
| Firefox | ✓ | ✓ | ✓ |
| Chrome (iOS) | ✗ | — | — |
