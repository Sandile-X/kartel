# Scroll-Driven Frame Animation — Fix Reference
## Kota Store Website | March 2026

> **Purpose:** A complete record of every bug encountered and every fix applied to get the scroll-driven
> kota explosion animation working on the Kota Store website. Keep this for future reference.

---

## The Correct Working Implementation (Summary)

The animation works by:
1. A tall `<section id="scroll-anim">` (600vh) acts as the scroll spacer
2. A `position: fixed` panel `#scroll-anim-sticky` sits over the viewport, hidden by default
3. `IntersectionObserver` watches the spacer — fades the fixed panel in when the spacer enters view, fades it out when it leaves
4. `window` scroll events map `-rect.top / (sectionHeight - viewportHeight)` to a 0–1 progress value
5. 120 preloaded JPEG frames are drawn onto a full-viewport canvas in sequence
6. Canvas uses `mix-blend-mode: multiply` so white video backgrounds become transparent

---

## Bug 1 — The Nuclear Bug: `overflow-x: hidden` kills everything

**Root cause of all animation failures.**

### What it does
Setting `overflow-x: hidden` on `<html>` or `<body>` triggers a CSS spec rule: when you set
`overflow` on one axis, the other axis implicitly defaults to `auto`. This makes `<body>`
become the **scroll container** instead of the viewport (`window`).

### What breaks
- `window.scrollY` is always `0` — scroll progress is always 0 — always frame 0
- `window` scroll events **never fire** — `onScroll()` never runs
- `position: fixed` elements are "fixed" relative to body, so they scroll with the page
- The animation section appears as one static image that you scroll right past, followed by a huge empty black/white space

### The Fix
Replace `overflow-x: hidden` with `overflow-x: clip` on **both** `html` and `body`:

```css
html { overflow-x: clip; }
body { overflow-x: clip; }
```

`overflow-x: clip` clips overflow identically visually but **never creates a new scroll container**.
No stacking context, no axis-coupling, no breaking of `position: fixed`. This is the correct
production-safe way to prevent horizontal scrollbars.

---

## Bug 2 — `position: sticky` silently does nothing

**First thing attempted. Should never be used for this pattern.**

### What it does
`position: sticky` is broken by any ONE of these being present anywhere in the ancestor chain:
- `overflow-x: hidden` or `overflow: hidden` on any ancestor (see Bug 1)
- Tailwind CSS CDN preflight
- AOS (Animate On Scroll) library
- `content-visibility: auto` on a parent
- Any `overflow` value other than `visible` on a parent

### What it looks like
Panel scrolls off screen. Animation lifts slightly and disappears. Only 1–2 frames visible.

### The Fix
Use `position: fixed` instead. Show/hide using IntersectionObserver:

```css
#scroll-anim-sticky {
  position: fixed;          /* NOT sticky */
  top: 0; left: 0;
  width: 100%; height: 100vh;
  opacity: 0;
  visibility: hidden;
  pointer-events: none;
  transition: opacity 0.65s ease, visibility 0s linear 0.65s; /* delayed-visibility trick */
}

#scroll-anim-sticky.sh-active {
  opacity: 1;
  visibility: visible;
  pointer-events: auto;
  transition: opacity 0.65s ease, visibility 0s linear 0s;
}
```

```js
var io = new IntersectionObserver(function(entries) {
  if (entries[0].isIntersecting) {
    sticky.classList.add('sh-active');
  } else {
    sticky.classList.remove('sh-active');
  }
}, { threshold: 0 });
io.observe(section); // section = #scroll-anim spacer div
```

---

## Bug 3 — Canvas DPR scaling makes image tiny

### What it does
Using `canvas.width = window.innerWidth * window.devicePixelRatio` on a Retina screen
makes the canvas 2–3× the viewport size at pixel level. The image is drawn in the
top-left corner — the rest of the canvas is blank. On CSS it "looks right" but the
drawing coordinates are completely wrong.

### The Fix (Option A — simple, used here)
Set canvas to exact viewport pixels, no DPR:
```js
canvas.width  = window.innerWidth;
canvas.height = window.innerHeight;
```

### The Fix (Option B — Retina-aware)
Apply `ctx.setTransform(dpr, 0, 0, dpr, 0, 0)` after scaling, and use logical pixel
coordinates in all draw calls.

---

## Bug 4 — Canvas positioned wrong (not full-bleed)

### What it does
`max-width: 1280px; margin: auto; display: block` turns the canvas into a centered box.
White/empty space appears below or around the canvas.

### The Fix
```css
#scroll-anim-sticky {
  position: fixed; top: 0; left: 0; width: 100%; height: 100vh;
  overflow: hidden;
}
#frame-canvas {
  position: absolute; top: 0; left: 0;
  width: 100%; height: 100%;
  display: block;
}
```

---

## Bug 5 — Cover-fit drawing broken by DPR math

### What it does
Complex cover/contain math using DPR-multiplied dimensions produces wrong scale
values. Image draws tiny in the corner or stretched incorrectly.

### The Fix
Simple cover-fit with natural pixel values:
```js
function drawFrame(index) {
  var img = frames[index];
  if (!img || !canvasW) return;
  var iw    = img.naturalWidth  || 1280;
  var ih    = img.naturalHeight || 720;
  var scale = Math.max(canvasW / iw, canvasH / ih);  // cover (not contain)
  var dw = iw * scale, dh = ih * scale;
  var dx = (canvasW - dw) / 2, dy = (canvasH - dh) / 2;
  ctx.clearRect(0, 0, canvasW, canvasH);
  ctx.drawImage(img, dx, dy, dw, dh);
}
```

---

## Bug 6 — Scroll progress calculates against wrong container

### What it does
Using `window.scrollY - section.offsetTop` is inaccurate when the page has
transforms, sticky headers offsetting layout, or reflows. Progress jumps,
gets stuck at 0, or exceeds 1.0.

### The Fix
Use `getBoundingClientRect()` — always accurate regardless of layout:
```js
var rect     = section.getBoundingClientRect();
var sectionH = section.offsetHeight - window.innerHeight;
var progress = Math.max(0, Math.min(1, -rect.top / sectionH));
```
`-rect.top` = how many pixels the section's top has scrolled above the viewport top.

---

## Bug 7 — `var` vs `let` / Temporal Dead Zone crash

### What it does
`let currentFrame` declared after `resizeCanvas()` definition. `resizeCanvas()` calls
`drawFrame(currentFrame)`. If `resizeCanvas()` is called before the `let` declaration
is reached in execution, TDZ throws `Cannot access 'currentFrame' before initialization`.

### The Fix
Declare `var currentFrame = 0` (or `let`) **before** defining `resizeCanvas()`:
```js
var currentFrame = 0;   // MUST be before resizeCanvas

function resizeCanvas() { ... drawFrame(currentFrame); }
```

---

## Bug 8 — Scroll snap `body.overflow = 'hidden'` breaks animation

### What it does
Snap-stop feature calls `document.body.style.overflow = 'hidden'` to hold scroll.
But since `body` might not be the scroll container (or other elements are in play),
this either does nothing or permanently locks the page.

### The Fix
Remove snap-stop entirely. The animation works perfectly without it.
If snap is needed in future, target the actual scroll container — not `document.body`.

---

## Bug 9 — Scroll progress bar uses `width` but JS sets `scaleX`

### What it does
CSS had `width: 0%` with `transition: width`, but JS was setting
`element.style.transform = 'scaleX(x)'`. One of them was always ignored.

### The Fix
Pick one approach and apply it consistently. The `scaleX` approach is smoother:
```css
#scroll-progress {
  transform-origin: left;
  transform: scaleX(0);
  transition: transform 0.05s linear;
  /* no width property */
}
```
```js
progEl.style.transform = 'scaleX(' + progress + ')';
```

---

## Bug 10 — White video background blocks background text

### What it does
The kota explosion video was shot against a white background (standard studio setup).
When drawn on a white canvas, you see only white — no background text shows through.

### The Fix
`mix-blend-mode: multiply` on the canvas element:
```css
#frame-canvas {
  mix-blend-mode: multiply;
}
```
`multiply` blend mode: white (255,255,255) multiplied against anything = that thing.
White becomes transparent. The kota (amber/brown tones) remains fully visible.
Background elements (text, color fields) now show through the white areas.

---

## Final Working Code Structure

### HTML
```html
<section id="scroll-anim">                    <!-- 600vh spacer -->
  <div id="scroll-anim-sticky">               <!-- position: fixed, opacity: 0 default -->
    <div class="scroll-bg-text">KOTA</div>    <!-- z-index: 0, decorative -->
    <canvas id="frame-canvas"></canvas>        <!-- position: absolute, mix-blend-mode: multiply -->
    <!-- annotation cards -->
    <div class="section-label" id="scroll-label">SCROLL TO BUILD YOUR KOTA</div>
  </div>
</section>
```

### CSS Critical Rules
```css
html, body { overflow-x: clip; }  /* NOT hidden */

#scroll-anim { height: 600vh; position: relative; }

#scroll-anim-sticky {
  position: fixed; top: 0; left: 0; /* NOT sticky */
  width: 100%; height: 100vh;
  overflow: hidden; background: #fff;
  z-index: 99;
  opacity: 0; visibility: hidden; pointer-events: none;
  transition: opacity 0.65s ease, visibility 0s linear 0.65s;
}
#scroll-anim-sticky.sh-active {
  opacity: 1; visibility: visible; pointer-events: auto;
  transition: opacity 0.65s ease, visibility 0s linear 0s;
}
#frame-canvas {
  position: absolute; top: 0; left: 0;
  width: 100%; height: 100%; display: block;
  mix-blend-mode: multiply;
  z-index: 1;
}
```

### JS Critical Rules
```js
var currentFrame = 0;  // declare BEFORE resizeCanvas

function resizeCanvas() {
  canvasW = window.innerWidth;
  canvasH = window.innerHeight;
  canvas.width = canvasW;
  canvas.height = canvasH;
  drawFrame(currentFrame);
}

// IntersectionObserver (not scroll event) controls show/hide
var io = new IntersectionObserver(function(entries) {
  sticky.classList.toggle('sh-active', entries[0].isIntersecting);
}, { threshold: 0 });
io.observe(section);

// Scroll progress via getBoundingClientRect
var rect = section.getBoundingClientRect();
var progress = Math.max(0, Math.min(1, -rect.top / (section.offsetHeight - window.innerHeight)));
```

---

## Frame Extraction Command (FFmpeg)

120 frames from an 8-second video at 15fps:
```powershell
ffmpeg -i "videos/Kota_exploding.mp4" -vf "fps=15" -q:v 3 "frames/frame_%04d.jpg"
```

Verify: `(Get-ChildItem "frames/frame_*.jpg").Count` → should return `120`

Frames served at `frames/frame_0001.jpg` through `frames/frame_0120.jpg` relative to `index.html`.

---

## Local Server

Python HTTP server (must NOT open `file://` directly):
```powershell
cd "C:\Users\SANDILE\Desktop\kartel"
python -m http.server 8080
```
Open: `http://localhost:8080`

---

*Last updated: March 2026 — Kota Store homepage.*
