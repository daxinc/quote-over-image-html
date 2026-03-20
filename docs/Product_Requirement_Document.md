add# Product Requirement Document (PRD): Quote Over Image

## 1. Product Overview
**Quote Over Image** is a lightweight, browser-based utility that allows users to overlay custom text onto curated background images. The primary goal is to provide a seamless "design-to-download" experience without requiring server-side processing or design expertise.

---

## 2. Page Layout

The application is a single HTML page with a two-region layout: a **Preview Area** on the left and a **Controls Panel** on the right.

### 2.1 Overall Structure (Desktop)

```
┌─────────────────────────────────────────────────────────────┐
│                        Page Header                          │
│                  "Quote Over Image" + tagline               │
├────────────────────────────────┬────────────────────────────┤
│                                │   ┌──────────────────────┐ │
│                                │   │   Image Gallery      │ │
│                                │   │   (thumbnail grid)   │ │
│                                │   └──────────────────────┘ │
│       Preview Canvas           │   ┌──────────────────────┐ │
│    (background + text overlay) │   │   Text Input         │ │
│                                │   │   (multiline)        │ │
│         Fixed 4:3 ratio        │   └──────────────────────┘ │
│                                │   ┌──────────────────────┐ │
│                                │   │   Font Size Controls │ │
│                                │   │   [ - ] [Reset] [ + ]│ │
│                                │   └──────────────────────┘ │
│                                │   ┌──────────────────────┐ │
│                                │   │   Alignment Controls │ │
│                                │   │   H: [L] [C] [R]     │ │
│                                │   │   V: [T] [M] [B]     │ │
│                                │   └──────────────────────┘ │
│                                │   ┌──────────────────────┐ │
│                                │   │   [ Download PNG ]   │ │
│                                │   └──────────────────────┘ │
├────────────────────────────────┴────────────────────────────┤
│                        Page Footer                          │
└─────────────────────────────────────────────────────────────┘
```

- **Left column (60% width):** Preview canvas, vertically centered.
- **Right column (40% width):** Controls panel, stacked top-to-bottom in the order shown above.

### 2.2 Responsive Behavior (Mobile / Narrow Viewport)
On viewports narrower than **768px**, the layout collapses to a single column:
1. Preview canvas (full width, maintains 4:3 aspect ratio)
2. Controls panel (full width, same stacking order)

---

## 3. Feature Specifications

### 3.1 Preview Canvas
- A `<div>` that acts as the real-time design canvas.
- Displays the selected background image via `background-image` (cover, centered).
- Overlays user-inputted text, reflecting alignment and font size changes instantly.
- Maintains a **4:3 aspect ratio** using `aspect-ratio: 4/3` or a padding-based fallback.
- The text overlay is positioned inside the canvas using flexbox, with padding (~5%) to keep text away from edges.

### 3.2 Image Gallery
- A horizontally scrollable row of clickable thumbnail images.
- Clicking a thumbnail updates the preview canvas background.
- The currently selected thumbnail has a visible highlight (e.g., a colored border).
- Ships with 4–6 pre-bundled images optimized for web (JPEG, ≤200 KB each).

### 3.3 Text Input
- A `<textarea>` for multiline quote entry.
- Placeholder text: *"Type your quote here..."*
- Changes are reflected on the preview canvas in real time (on `input` event).

### 3.4 Font Size Controls
Three inline buttons:
| Button | Action |
|--------|--------|
| `[ - ]` | Decrease font size by 2px (minimum: 12px) |
| `[ Reset ]` | Reset to default size (24px) |
| `[ + ]` | Increase font size by 2px (maximum: 72px) |

The current font size value should be displayed next to the controls (e.g., "24px").

### 3.5 Alignment Controls
Two independent button groups that combine to produce 9 possible positions:

**Horizontal (3 toggle buttons):** Left | Center | Right
**Vertical (3 toggle buttons):** Top | Middle | Bottom

Default alignment: **Center + Middle** (text centered in both axes).

Within each group, exactly one option is active at a time (radio-button behavior). The active button should have a visually distinct style.

### 3.6 Export (Download)
- A prominent "Download PNG" button.
- Uses `html2canvas` or `dom-to-image` to capture the preview canvas DOM element as a PNG.
- Runs entirely client-side — no server round-trip.
- Triggers an automatic browser download with a filename like `quote-over-image.png`.

---

## 4. Technical Requirements & Constraints

### 4.1 Stack
- **Single HTML file** with inline CSS and a `<script type="module">` block (no build step).
- **Preact + htm** (~5 KB gzip) for reactive UI — React's component/hooks API via tagged template literals, loaded as ESM from CDN.
- **html2canvas** for DOM-to-PNG export, loaded via CDN.

### 4.2 Browser Support
Modern "evergreen" browsers:
- Google Chrome (desktop & mobile)
- Mozilla Firefox
- Apple Safari
- Microsoft Edge

### 4.3 Text Legibility — Three-Layer Halo Rendering

Text must remain readable on any background — dark, light, or busy. This is achieved with a three-layer rendering stack, all implementable in CSS.

#### Layer 1: Backdrop Blur (busy backgrounds)
A frosted-glass effect behind the text region suppresses high-frequency detail that competes with letterforms.

```css
.text-overlay-backdrop {
  backdrop-filter: blur(2px);
  -webkit-backdrop-filter: blur(2px);
  border-radius: 8px;
}
```

> **Note:** `backdrop-filter` is supported in all evergreen browsers. This layer is applied to the text's container element, not to the text itself.

#### Layer 2: Halo Stroke (primary legibility layer)
Three stacked `text-shadow` passes produce a depth-graduated glow — tight at the edges, soft further out:

```css
.text-overlay {
  text-shadow:
    /* Pass A — tight crisp edge */
    0 0 1px rgba(0, 0, 0, 0.80),
    0 0 1px rgba(0, 0, 0, 0.80),
    /* Pass B — mid-range fill */
    0 0 3px rgba(0, 0, 0, 0.50),
    0 0 3px rgba(0, 0, 0, 0.50),
    /* Pass C — soft outer halo */
    0 0 6px rgba(0, 0, 0, 0.30),
    0 0 6px rgba(0, 0, 0, 0.30);
}
```

Duplicated values at each radius strengthen the shadow since CSS composites them additively.

#### Layer 3: Text Fill
White text at full opacity, rendered on top of the halo:

```css
.text-overlay {
  color: #ffffff;
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Helvetica, Arial, sans-serif;
}
```

#### Automatic Color Inversion (requires JS)
For backgrounds where white-on-dark doesn't work, an optional JS enhancement can flip text and halo colors:

1. Draw the current background image to an offscreen `<canvas>`.
2. Sample pixel luminance in the text bounding box using `getImageData()`.
3. Compute mean luminance: `L = (0.299R + 0.587G + 0.114B) / 255`.
4. If `L ≥ 0.5` (light background): set text to black (`#000000`), halo to white `rgba(255,255,255,…)`.
5. If `L < 0.5` (dark background): keep text white, halo black (default).

This is a progressive enhancement — the default white-text + dark-halo works well on most photographic backgrounds without it.

### 4.4 Performance
- No server-side processing.
- All images pre-bundled or loaded from static URLs.
- Total page weight target: under 1 MB (excluding background images).
