# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the App

No build step required. Open `index.html` directly in a browser, or serve it with any static file server:

```bash
python3 -m http.server 8080
# then open http://localhost:8080
```

## Architecture

This is a **zero-build, single-file web app**. All application logic lives in `index.html` as an inline `<script type="module">`. There is no `package.json`, no bundler, and no `node_modules`.

### File Structure

- `index.html` — entire application: inline CSS + Preact component tree
- `js/images.js` — declares the global `IMAGES` array (loaded via `<script src>` before the module script)
- `img/` — background images (`*.png`) and thumbnails (`*-thumbnail.png`)

### UI Framework

Uses **Preact + htm** loaded from CDN (`esm.sh`). htm provides JSX-like syntax via tagged template literals — no transpilation needed. The API is identical to React hooks (`useState`, `useRef`, `useCallback`, `useEffect`).

```js
import { h, render } from 'https://esm.sh/preact@10';
import { useState, useRef, useCallback, useEffect } from 'https://esm.sh/preact@10/hooks';
import htm from 'https://esm.sh/htm@3';
const html = htm.bind(h);
```

### Component Tree

All state lives in `App`. Child components are stateless props receivers:

```
App
├── PreviewCanvas     — background image div + text overlay span (receives canvasRef)
├── ImageGallery      — horizontally scrollable thumbnail row
├── TextInput         — textarea for quote text
├── FontSizeControls  — −/+/Reset buttons + direct numeric input; supports scroll wheel
├── AlignmentControls — H (left/center/right) and V (top/middle/bottom) toggle buttons
└── DownloadButton    — triggers html2canvas export
```

### Export Pipeline

`DownloadButton` uses `html2canvas@1.4.1` (loaded from jsdelivr CDN) to capture the `.canvas` DOM element at `scale: 2` (retina) and triggers a PNG download via `URL.createObjectURL`. Known limitation: `backdrop-filter` blur is not rendered by html2canvas in the exported PNG.

### Text Legibility

Text readability on varied backgrounds is achieved with three CSS layers:
1. `backdrop-filter: blur(2px)` on the text container (suppressed in export)
2. Stacked `text-shadow` halo (six passes at radii 1px/3px/6px) on the text span
3. White text fill (`color: #ffffff`); an `inverted` CSS class switches to black text with white halo

### Adding Images

Add entries to `js/images.js` in the `IMAGES` array with `{ id, src, thumb }`. Place full-size images at `img/<id>.png` and thumbnails at `img/<id>-thumbnail.png`. The canvas aspect ratio is derived dynamically from the selected image's natural dimensions via `useEffect`.

## gstack

Use the `/browse` skill from gstack for all web browsing. Never use `mcp__claude-in-chrome__*` tools directly.

Available skills:
`/office-hours`, `/plan-ceo-review`, `/plan-eng-review`, `/plan-design-review`, `/design-consultation`, `/review`, `/ship`, `/browse`, `/qa`, `/qa-only`, `/design-review`, `/setup-browser-cookies`, `/retro`, `/investigate`, `/document-release`, `/codex`, `/careful`, `/freeze`, `/guard`, `/unfreeze`, `/gstack-upgrade`
