# Technical Design Document: Quote Over Image

> **Reference:** [Product_Requirement_Document.md](./Product_Requirement_Document.md)

---

## 1. Architecture Overview

The application ships as a **single `index.html` file** with inline `<style>` and a `<script type="module">` block. It uses **Preact + htm** for reactive UI rendering — React's component model without a build step.

```
index.html
├── <style>           — all CSS (layout, components, responsive, halo rendering)
├── <div id="app">    — Preact mount point
└── <script type="module">
    ├── import { html, render } from htm/preact/standalone  (CDN)
    ├── import { useState, useRef, useCallback } from preact/hooks  (CDN)
    ├── Component definitions (App, PreviewCanvas, Gallery, Controls…)
    └── render(html`<${App} />`, document.getElementById("app"))
```

No build step, no bundler, no `node_modules`.

### 1.1 External Dependencies

| Dependency | Size (gzip) | Purpose | Load method |
|---|---|---|---|
| Preact | ~4 KB | Virtual DOM, components, hooks | ESM from `esm.sh` |
| htm | ~1 KB | Tagged template JSX alternative | ESM from `esm.sh` (bundled with Preact standalone) |
| html2canvas | ^1.4.1 | DOM-to-PNG capture | `<script>` from `cdn.jsdelivr.net` |

Total framework overhead: **~5 KB gzipped**.

### 1.2 Why Preact + htm

- **No build step** — `htm` uses tagged template literals instead of JSX, so the browser runs it directly.
- **React-compatible API** — `useState`, `useRef`, `useCallback`, `useEffect` all work identically.
- **Single ESM import** — `htm/preact/standalone` bundles everything into one URL.
- **Transferable knowledge** — anyone who knows React can read and modify this code.

---

## 2. Component Tree

```
App
├── Header
├── PreviewCanvas        ← receives: backgroundUrl, quoteText, fontSize, alignH, alignV, canvasRef
│   └── .text-overlay-backdrop
│       └── .text-overlay
├── ImageGallery         ← receives: images, selectedId, onSelect
│   └── Thumbnail × N
├── TextInput            ← receives: value, onInput
├── FontSizeControls     ← receives: fontSize, onChange
├── AlignmentControls    ← receives: alignH, alignV, onChangeH, onChangeV
└── DownloadButton       ← receives: canvasRef
```

All state lives in `App` via `useState` hooks. Child components are pure — they receive props and call callbacks. No shared context needed for this scale.

---

## 3. CSS Architecture

CSS is defined in an inline `<style>` block. It is **not** managed by Preact — components reference class names directly.

### 3.1 Layout System

```css
.workspace {
  display: grid;
  grid-template-columns: 3fr 2fr;
  gap: 24px;
  max-width: 1200px;
  margin: 0 auto;
  padding: 24px;
}

@media (max-width: 768px) {
  .workspace {
    grid-template-columns: 1fr;
  }
}
```

### 3.2 Canvas Aspect Ratio

```css
.canvas {
  position: relative;
  width: 100%;
  aspect-ratio: 4 / 3;
  background-size: cover;
  background-position: center;
  overflow: hidden;
  border-radius: 8px;
}
```

### 3.3 Text Positioning via Flexbox

Alignment is driven by inline `style` set by the component, not by toggling CSS classes. This is simpler in a Preact model — the component computes the `justify-content` and `align-items` values from props.

```css
.text-overlay-backdrop {
  position: absolute;
  inset: 0;
  display: flex;
  padding: 5%;
  backdrop-filter: blur(2px);
  -webkit-backdrop-filter: blur(2px);
}
```

Alignment mapping (computed in JS, applied as inline style):

| `alignH` prop | `justifyContent` value |
|---|---|
| `"left"` | `"flex-start"` |
| `"center"` | `"center"` |
| `"right"` | `"flex-end"` |

| `alignV` prop | `alignItems` value |
|---|---|
| `"top"` | `"flex-start"` |
| `"middle"` | `"center"` |
| `"bottom"` | `"flex-end"` |

### 3.4 Halo Rendering (CSS Layers 2 + 3)

```css
.text-overlay {
  color: #ffffff;
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Helvetica, Arial, sans-serif;
  line-height: 1.4;
  text-align: center;
  white-space: pre-wrap;
  word-break: break-word;
  max-width: 90%;
  text-shadow:
    0 0 1px rgba(0, 0, 0, 0.80),
    0 0 1px rgba(0, 0, 0, 0.80),
    0 0 3px rgba(0, 0, 0, 0.50),
    0 0 3px rgba(0, 0, 0, 0.50),
    0 0 6px rgba(0, 0, 0, 0.30),
    0 0 6px rgba(0, 0, 0, 0.30);
}

.text-overlay.inverted {
  color: #000000;
  text-shadow:
    0 0 1px rgba(255, 255, 255, 0.80),
    0 0 1px rgba(255, 255, 255, 0.80),
    0 0 3px rgba(255, 255, 255, 0.50),
    0 0 3px rgba(255, 255, 255, 0.50),
    0 0 6px rgba(255, 255, 255, 0.30),
    0 0 6px rgba(255, 255, 255, 0.30);
}
```

### 3.5 Controls Panel Styling

```css
.controls-panel {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.control-group {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  align-items: center;
}

.btn {
  padding: 8px 16px;
  border: 1px solid #d1d5db;
  border-radius: 6px;
  background: #ffffff;
  cursor: pointer;
  font-size: 14px;
  transition: background-color 0.15s, border-color 0.15s;
}

.btn.active {
  background: #3b82f6;
  color: #ffffff;
  border-color: #3b82f6;
}

.btn-primary {
  background: #3b82f6;
  color: #ffffff;
  border-color: #3b82f6;
  width: 100%;
  padding: 12px;
  font-size: 16px;
  font-weight: 600;
}
```

---

## 4. Component Implementations

### 4.1 App (Root)

```javascript
import { html, render } from 'https://esm.sh/htm/preact/standalone';
import { useState, useRef, useCallback } from 'https://esm.sh/preact/hooks';

const FONT_MIN = 12;
const FONT_MAX = 72;
const FONT_DEFAULT = 24;
const FONT_STEP = 2;

function App() {
  const [backgroundUrl, setBackgroundUrl] = useState(IMAGES[0].src);
  const [selectedId, setSelectedId]       = useState(IMAGES[0].id);
  const [quoteText, setQuoteText]         = useState("");
  const [fontSize, setFontSize]           = useState(FONT_DEFAULT);
  const [alignH, setAlignH]              = useState("center");
  const [alignV, setAlignV]              = useState("middle");
  const canvasRef = useRef(null);

  const handleSelectImage = useCallback((img) => {
    setBackgroundUrl(img.src);
    setSelectedId(img.id);
  }, []);

  const handleFontSize = useCallback((delta) => {
    if (delta === 0) return setFontSize(FONT_DEFAULT);
    setFontSize(s => Math.min(FONT_MAX, Math.max(FONT_MIN, s + delta)));
  }, []);

  return html`
    <header class="header">
      <h1>Quote Over Image</h1>
      <p class="tagline">Create beautiful quote images in seconds</p>
    </header>
    <main class="workspace">
      <section class="preview-section">
        <${PreviewCanvas}
          backgroundUrl=${backgroundUrl}
          quoteText=${quoteText}
          fontSize=${fontSize}
          alignH=${alignH}
          alignV=${alignV}
          canvasRef=${canvasRef}
        />
      </section>
      <aside class="controls-panel">
        <${ImageGallery}
          images=${IMAGES}
          selectedId=${selectedId}
          onSelect=${handleSelectImage}
        />
        <${TextInput} value=${quoteText} onInput=${setQuoteText} />
        <${FontSizeControls} fontSize=${fontSize} onChange=${handleFontSize} />
        <${AlignmentControls}
          alignH=${alignH} alignV=${alignV}
          onChangeH=${setAlignH} onChangeV=${setAlignV}
        />
        <${DownloadButton} canvasRef=${canvasRef} />
      </aside>
    </main>
  `;
}

render(html`<${App} />`, document.getElementById("app"));
```

### 4.2 PreviewCanvas

```javascript
const ALIGN_MAP = {
  left: "flex-start", center: "center", right: "flex-end",
  top: "flex-start",  middle: "center", bottom: "flex-end",
};

function PreviewCanvas({ backgroundUrl, quoteText, fontSize, alignH, alignV, canvasRef }) {
  const backdropStyle = {
    justifyContent: ALIGN_MAP[alignH],
    alignItems: ALIGN_MAP[alignV],
  };

  return html`
    <div
      class="canvas"
      ref=${canvasRef}
      style="background-image: url('${backgroundUrl}')"
    >
      <div class="text-overlay-backdrop" style=${backdropStyle}>
        ${quoteText && html`
          <span class="text-overlay" style="font-size: ${fontSize}px">
            ${quoteText}
          </span>
        `}
      </div>
    </div>
  `;
}
```

The conditional `${quoteText && …}` ensures no empty `<span>` is rendered when the input is blank.

### 4.3 ImageGallery

```javascript
function ImageGallery({ images, selectedId, onSelect }) {
  return html`
    <div class="control-group gallery">
      ${images.map(img => html`
        <img
          key=${img.id}
          class="thumbnail ${img.id === selectedId ? "selected" : ""}"
          src=${img.thumb}
          alt=${img.id}
          onClick=${() => onSelect(img)}
        />
      `)}
    </div>
  `;
}
```

### 4.4 FontSizeControls

```javascript
function FontSizeControls({ fontSize, onChange }) {
  return html`
    <div class="control-group">
      <button class="btn" onClick=${() => onChange(-FONT_STEP)}>−</button>
      <span class="font-size-display">${fontSize}px</span>
      <button class="btn" onClick=${() => onChange(0)}>Reset</button>
      <button class="btn" onClick=${() => onChange(FONT_STEP)}>+</button>
    </div>
  `;
}
```

### 4.5 AlignmentControls

```javascript
function AlignmentControls({ alignH, alignV, onChangeH, onChangeV }) {
  const hOptions = ["left", "center", "right"];
  const vOptions = ["top", "middle", "bottom"];

  return html`
    <div class="control-group">
      <span class="control-label">H:</span>
      ${hOptions.map(v => html`
        <button
          key=${v}
          class="btn ${alignH === v ? "active" : ""}"
          onClick=${() => onChangeH(v)}
        >${v.charAt(0).toUpperCase()}</button>
      `)}
    </div>
    <div class="control-group">
      <span class="control-label">V:</span>
      ${vOptions.map(v => html`
        <button
          key=${v}
          class="btn ${alignV === v ? "active" : ""}"
          onClick=${() => onChangeV(v)}
        >${v.charAt(0).toUpperCase()}</button>
      `)}
    </div>
  `;
}
```

### 4.6 DownloadButton

```javascript
function DownloadButton({ canvasRef }) {
  const [exporting, setExporting] = useState(false);

  const handleExport = useCallback(async () => {
    if (!canvasRef.current || exporting) return;
    setExporting(true);
    try {
      const rendered = await html2canvas(canvasRef.current, {
        useCORS: true,
        scale: 2,
        backgroundColor: null,
        logging: false,
      });
      rendered.toBlob(blob => {
        const url = URL.createObjectURL(blob);
        const a = document.createElement("a");
        a.href = url;
        a.download = "quote-over-image.png";
        a.click();
        URL.revokeObjectURL(url);
      }, "image/png");
    } finally {
      setExporting(false);
    }
  }, [canvasRef, exporting]);

  return html`
    <button
      class="btn btn-primary"
      onClick=${handleExport}
      disabled=${exporting}
    >
      ${exporting ? "Exporting…" : "Download PNG"}
    </button>
  `;
}
```

---

## 5. Image Gallery Data

```javascript
const IMAGES = [
  { id: "mountain", src: "images/mountain.jpg", thumb: "images/mountain_thumb.jpg" },
  { id: "ocean",    src: "images/ocean.jpg",    thumb: "images/ocean_thumb.jpg"    },
  { id: "forest",   src: "images/forest.jpg",   thumb: "images/forest_thumb.jpg"   },
  { id: "city",     src: "images/city.jpg",     thumb: "images/city_thumb.jpg"     },
  { id: "desert",   src: "images/desert.jpg",   thumb: "images/desert_thumb.jpg"   },
];
```

Alternatively, `src` and `thumb` can point to external URLs (e.g., Unsplash). The first image is selected by default.

### 5.1 Gallery CSS

```css
.gallery {
  overflow-x: auto;
  padding-bottom: 4px;
  flex-wrap: nowrap;
}

.thumbnail {
  width: 72px;
  height: 54px;
  object-fit: cover;
  border-radius: 6px;
  cursor: pointer;
  border: 2px solid transparent;
  transition: border-color 0.15s;
  flex-shrink: 0;
}

.thumbnail.selected {
  border-color: #3b82f6;
}
```

---

## 6. Export Pipeline

### 6.1 Sequence

```
User clicks "Download PNG"
  → DownloadButton sets exporting=true (button disabled, shows "Exporting…")
  → html2canvas(canvasRef.current, { useCORS: true, scale: 2 })
  → returns Canvas element
  → canvas.toBlob("image/png")
  → create Object URL → programmatic <a> click → revoke URL
  → DownloadButton sets exporting=false
```

### 6.2 html2canvas Configuration

| Option | Value | Reason |
|---|---|---|
| `useCORS` | `true` | Allow cross-origin images |
| `scale` | `2` | Retina-quality output |
| `backgroundColor` | `null` | Preserve transparency |
| `logging` | `false` | Suppress console noise |

### 6.3 Known Limitation: `backdrop-filter`

`html2canvas` does **not** render `backdrop-filter`. The 2px backdrop blur will be absent from exported PNGs. The text-shadow halo alone provides sufficient legibility. Accepted for v1.

---

## 7. Luminance Sampling (Color Inversion)

Progressive enhancement — not a v1 blocker.

### 7.1 Algorithm

```javascript
function sampleLuminance(imageUrl, region) {
  return new Promise(resolve => {
    const img = new Image();
    img.crossOrigin = "anonymous";
    img.onload = () => {
      const c = document.createElement("canvas");
      c.width = img.naturalWidth;
      c.height = img.naturalHeight;
      const ctx = c.getContext("2d");
      ctx.drawImage(img, 0, 0);

      const data = ctx.getImageData(region.x, region.y, region.w, region.h).data;
      let sum = 0;
      const pixelCount = data.length / 4;
      for (let i = 0; i < data.length; i += 4) {
        sum += 0.299 * data[i] + 0.587 * data[i + 1] + 0.114 * data[i + 2];
      }
      resolve(sum / pixelCount / 255);
    };
    img.src = imageUrl;
  });
}
```

### 7.2 Integration with Preact

Luminance sampling would be triggered via `useEffect` in `PreviewCanvas`, reacting to changes in `backgroundUrl`, `alignH`, or `alignV`:

```javascript
const [inverted, setInverted] = useState(false);

useEffect(() => {
  if (!canvasRef.current) return;
  const rect = canvasRef.current.getBoundingClientRect();
  const region = mapToImageCoords(rect, alignH, alignV);
  sampleLuminance(backgroundUrl, region).then(lum => {
    setInverted(lum >= 0.5);
  });
}, [backgroundUrl, alignH, alignV]);
```

### 7.3 Region Mapping (background-size: cover)

```
canvasAspect = canvasWidth / canvasHeight
imageAspect  = imageNaturalWidth / imageNaturalHeight

if imageAspect > canvasAspect:
  visibleWidth  = imageNaturalHeight × canvasAspect
  offsetX       = (imageNaturalWidth − visibleWidth) / 2
  scaleX        = visibleWidth / canvasWidth
else:
  visibleHeight = imageNaturalWidth / canvasAspect
  offsetY       = (imageNaturalHeight − visibleHeight) / 2
  scaleY        = visibleHeight / canvasHeight
```

---

## 8. File & Asset Organization

```
quote-over-image/
├── index.html                      — single-file application
├── images/
│   ├── mountain.jpg                — 1600px wide, ≤200 KB
│   ├── mountain_thumb.jpg          — 150px wide
│   ├── ocean.jpg
│   ├── ocean_thumb.jpg
│   ├── forest.jpg
│   ├── forest_thumb.jpg
│   ├── city.jpg
│   ├── city_thumb.jpg
│   ├── desert.jpg
│   └── desert_thumb.jpg
├── Product_Requirement_Document.md
└── Technical_Design_Document.md
```

If using external image URLs, the `images/` directory is not needed.

---

## 9. Edge Cases & Constraints

| Scenario | Handling |
|---|---|
| Empty text input | `${quoteText && html\`…\`}` — no `<span>` rendered |
| Very long text | `word-break: break-word` + `max-width: 90%` wraps text; canvas clips overflow |
| Text exceeds canvas height | `overflow: hidden` on canvas; user must reduce font size |
| CORS on external images | `html2canvas` uses `useCORS: true`; images need CORS headers |
| `backdrop-filter` in export | Not rendered by `html2canvas`; accepted for v1 (§6.3) |
| `aspect-ratio` unsupported | Fallback: `padding-top: 75%` on `.canvas` |
| CDN unavailable | Feature-detect `html2canvas` on load; disable download button with tooltip if missing |
| ESM import failure | `<script type="module">` errors are silent; add a `<noscript>` fallback message |
