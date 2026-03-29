# Scroll-Driven Frame Animation — Implementation Guide

> **What this document is:** A complete technical reference for the phone deconstruction scroll animation on the Omegatek Solutions homepage. Use this to re-implement, debug, or hand off the feature to any developer or AI assistant.

---

## What the Feature Does

A full-screen canvas sits fixed over the viewport. As the user scrolls through a tall spacer section, a sequence of JPEG frames (extracted from a video) is drawn onto the canvas one-by-one — creating the illusion that the video plays in response to scroll. Floating annotation cards appear and disappear at specific points in the animation. The whole panel fades in when the scroll section enters the viewport and fades out when it leaves.

---

## Part 1 — Prerequisites & Setup

### 1.1 Extract frames from the source video using FFmpeg

You need a folder of numbered JPEG files. The target is **~96 frames** (12 fps from an 8-second clip is ideal — enough fidelity without being heavy).

**Windows FFmpeg path (WinGet install):**
```
C:\Users\SANDILE\AppData\Local\Microsoft\WinGet\Packages\Gyan.FFmpeg_Microsoft.Winget.Source_8wekyb3d8bbwe\ffmpeg-8.1-full_build\bin\ffmpeg.exe
```

**Extraction command:**
```powershell
& "C:\Users\SANDILE\AppData\Local\Microsoft\WinGet\Packages\Gyan.FFmpeg_Microsoft.Winget.Source_8wekyb3d8bbwe\ffmpeg-8.1-full_build\bin\ffmpeg.exe" `
  -i "images2\Phone_deconstruction_animation.mp4" `
  -vf "fps=12" `
  -q:v 3 `
  "demo\frames\frame_%04d.jpg"
```

- `-vf "fps=12"` — 12 frames per second of source video
- `-q:v 3` — JPEG quality (1=best, 31=worst; 3 is sharp enough, stays fast)
- `frame_%04d.jpg` — zero-padded 4-digit names: `frame_0001.jpg` … `frame_0096.jpg`

**Verify the output:**
```powershell
(Get-ChildItem "demo\frames\frame_*.jpg").Count   # should print 96
```

Frames must be served from the same origin as the page (no `file://` directly). Run a local PHP server:
```powershell
php -S 127.0.0.1:8080 router.php
```

---

## Part 2 — HTML Structure

Place this immediately after your hero section and before the next content section:

```html
<!-- sub-hero: phone deconstruction scroll animation -->
<div id="subhero-section">
  <div id="subhero-sticky">
    <canvas id="subhero-canvas"></canvas>
    <div id="subhero-label">Scroll to Deconstruct</div>

    <!-- Cards: data-show / data-hide are 0.0–1.0 scroll progress values -->
    <div class="sh-card" id="sh-card-1" data-show="0.10" data-hide="0.35">
      <div class="sh-num">01 — Screen</div>
      <div class="sh-title">Cracked Display</div>
      <div class="sh-desc">OLED/LCD panels replaced with OEM-grade parts. Restored in under 45 minutes.</div>
      <div class="sh-stat">45<span style="font-size:16px">min</span></div>
      <div class="sh-stat-label">Average repair time</div>
    </div>

    <div class="sh-card" id="sh-card-2" data-show="0.30" data-hide="0.55">
      <div class="sh-num">02 — Battery</div>
      <div class="sh-title">Power Cell</div>
      <div class="sh-desc">Genuine capacity batteries. 0–100% health back, no memory degradation.</div>
      <div class="sh-stat">100<span style="font-size:16px">%</span></div>
      <div class="sh-stat-label">Battery health restored</div>
    </div>

    <div class="sh-card" id="sh-card-3" data-show="0.52" data-hide="0.75">
      <div class="sh-num">03 — Logic Board</div>
      <div class="sh-title">Motherboard</div>
      <div class="sh-desc">Micro-soldering for water damage, no-power faults, and data recovery.</div>
      <div class="sh-stat">98<span style="font-size:16px">%</span></div>
      <div class="sh-stat-label">Board save rate</div>
    </div>

    <div class="sh-card" id="sh-card-4" data-show="0.72" data-hide="0.95">
      <div class="sh-num">04 — Cameras</div>
      <div class="sh-title">Optics Module</div>
      <div class="sh-desc">All lenses, sensors and OIS actuators restored. Crystal-clear shots guaranteed.</div>
      <div class="sh-stat">4K</div>
      <div class="sh-stat-label">Quality restored</div>
    </div>

    <div id="subhero-cta">
      <a href="#services" class="btn">get started</a>
    </div>
  </div>
</div>
```

**Key rule:** `#subhero-section` is the tall scroll spacer (600vh). `#subhero-sticky` is the panel that gets fixed. The canvas fills `#subhero-sticky` wall-to-wall.

---

## Part 3 — CSS

Add this block to your site's stylesheet. **Critical notes are inline.**

```css
/* ─── Sub-Hero Scroll Animation ──────────────────────────── */
#subhero-section {
  height: 600vh;        /* controls how long the animation lasts */
  position: relative;   /* required for IntersectionObserver to detect correctly */
}

/*
  CRITICAL: Use position:fixed — NOT position:sticky.

  position:sticky silently breaks in any page that has:
    - overflow-x:hidden on <html> or <body>
    - Tailwind CSS preflight
    - AOS (Animate On Scroll) library
    - content-visibility:auto on a parent
  Any ONE of these kills sticky. position:fixed is immune to all of them.

  The fixed panel is hidden by default and shown ONLY when the spacer
  section is in the viewport (controlled by IntersectionObserver in JS).
*/
#subhero-sticky {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100vh;
  overflow: hidden;
  background: #fff;
  z-index: 99;          /* below navbar (typically z-index:1000) */

  /* Smooth fade in/out — the delayed-visibility trick:
     On fade OUT: opacity transitions to 0 over 0.65s, THEN visibility:hidden
                  fires (delayed by 0.65s) so the fade is visible before hiding.
     On fade IN:  visibility:visible fires instantly (0s delay), THEN opacity fades in. */
  opacity: 0;
  visibility: hidden;
  pointer-events: none;
  transition: opacity 0.65s ease, visibility 0s linear 0.65s;
}

#subhero-sticky.sh-active {
  opacity: 1;
  visibility: visible;
  pointer-events: auto;
  transition: opacity 0.65s ease, visibility 0s linear 0s;
}

#subhero-canvas {
  position: absolute;
  top: 0; left: 0;
  width: 100%; height: 100%;
  display: block;
}

/* ─── Annotation Cards ───────────────────────────────────── */
/*
  IMPORTANT: If your site uses html { font-size: 62.5% } (common reset that
  makes 1rem = 10px), do NOT use rem units for card text — they will be
  tiny (7–10px). Use px units directly.
*/
.sh-card {
  position: absolute;
  background: rgba(255,255,255,0.96);
  backdrop-filter: blur(16px);
  -webkit-backdrop-filter: blur(16px);
  border: 1px solid rgba(179,12,230,0.25);
  border-left: 4px solid #b30ce6;   /* brand accent stripe */
  border-radius: 16px;
  padding: 20px 24px;
  width: 260px;
  max-width: 30vw;
  opacity: 0;
  transform: translateY(18px) scale(0.96);
  transition: opacity 0.45s ease, transform 0.45s ease;
  pointer-events: none;
  box-shadow: 0 8px 36px rgba(179,12,230,0.18), 0 2px 10px rgba(0,0,0,0.07);
  z-index: 2;
}

.sh-card.visible {
  opacity: 1;
  transform: translateY(0) scale(1);
}

.sh-card .sh-num   { font-size: 11px; font-weight: 700; letter-spacing: 0.12em; color: #b30ce6; text-transform: uppercase; margin-bottom: 6px; }
.sh-card .sh-title { font-size: 18px; font-weight: 700; color: #1a1a2e; margin-bottom: 8px; line-height: 1.2; }
.sh-card .sh-desc  { font-size: 13px; color: #555; line-height: 1.6; margin-bottom: 12px; }
.sh-card .sh-stat  { font-size: 36px; font-weight: 800; color: #b30ce6; line-height: 1; letter-spacing: -0.02em; }
.sh-card .sh-stat span { font-size: 16px; font-weight: 600; }
.sh-card .sh-stat-label { font-size: 12px; color: #999; margin-top: 4px; text-transform: uppercase; letter-spacing: 0.06em; }

/* Card positions — corners of the viewport */
#sh-card-1 { bottom: 20%; left:  4%; }
#sh-card-2 { top:    20%; right: 4%; }
#sh-card-3 { bottom: 20%; right: 4%; }
#sh-card-4 { top:    20%; left:  4%; }

/* ─── Scroll hint label ──────────────────────────────────── */
#subhero-label {
  position: absolute;
  top: 2rem; left: 50%;
  transform: translateX(-50%);
  background: rgba(179,12,230,0.08);
  border: 1px solid rgba(179,12,230,0.25);
  color: #b30ce6;
  padding: 0.3rem 1rem;
  border-radius: 999px;
  font-size: 13px;
  font-weight: 500;
  letter-spacing: 0.06em;
  text-transform: uppercase;
  white-space: nowrap;
  z-index: 2;
}

/* ─── CTA button (appears at end of animation) ───────────── */
#subhero-cta {
  position: absolute;
  bottom: 8%; left: 50%;
  transform: translateX(-50%);
  opacity: 0;
  pointer-events: none;
  transition: opacity 0.5s ease;
  z-index: 2;
  white-space: nowrap;
}
#subhero-cta.visible { opacity: 1; pointer-events: auto; }

/* ─── Mobile ─────────────────────────────────────────────── */
@media (max-width: 768px) {
  #subhero-section { height: 400vh; }
  .sh-card { width: 160px; max-width: 44vw; padding: 12px 14px; border-left-width: 3px; }
  .sh-card .sh-desc, .sh-card .sh-stat-label { display: none; }
  .sh-card .sh-title { font-size: 14px; }
  .sh-card .sh-stat  { font-size: 26px; }
  .sh-card .sh-num   { font-size: 10px; }
  #sh-card-1 { bottom: 3vh; left:  2%; }
  #sh-card-2 { top:    10vh; right: 2%; }
  #sh-card-3 { bottom: 3vh; right: 2%; }
  #sh-card-4 { top:    10vh; left: 2%; }
}
```

---

## Part 4 — JavaScript

Place this `<script>` block at the **very bottom of `<body>`**, just before `</body>`. It must come after all other scripts that might query DOM elements.

```html
<!-- Sub-Hero Scroll Animation -->
<script>
(function() {
  'use strict';

  var FRAME_COUNT = 96;
  var frames = new Array(FRAME_COUNT);
  var loadedCount = 0;

  // ── Canvas setup ────────────────────────────────────────
  var canvas = document.getElementById('subhero-canvas');
  var ctx    = canvas.getContext('2d');
  var canvasW = 0, canvasH = 0;
  var currentFrame = 0;   // MUST be declared before resizeCanvas is called

  function resizeCanvas() {
    canvasW = window.innerWidth;
    canvasH = window.innerHeight;
    canvas.width  = canvasW;
    canvas.height = canvasH;
    drawFrame(currentFrame);
  }

  // Cover-fit: fills the viewport, crops edges rather than letterboxing
  function drawFrame(index) {
    var img = frames[index];
    if (!img || !canvasW) return;
    var iw    = img.naturalWidth  || 1280;
    var ih    = img.naturalHeight || 720;
    var scale = Math.max(canvasW / iw, canvasH / ih);  // cover, not contain
    var dw    = iw * scale, dh = ih * scale;
    var dx    = (canvasW - dw) / 2, dy = (canvasH - dh) / 2;
    ctx.clearRect(0, 0, canvasW, canvasH);
    ctx.drawImage(img, dx, dy, dw, dh);
  }

  resizeCanvas();
  window.addEventListener('resize', resizeCanvas, { passive: true });

  // ── DOM references ───────────────────────────────────────
  var section = document.getElementById('subhero-section');
  var sticky  = document.getElementById('subhero-sticky');
  var shCards = document.querySelectorAll('.sh-card');
  var cta     = document.getElementById('subhero-cta');

  // ── IntersectionObserver: show/hide the fixed panel ─────
  //
  // When the spacer div (#subhero-section) scrolls into view → add .sh-active
  // When it scrolls out → remove .sh-active (CSS handles the fade)
  //
  var io = new IntersectionObserver(function(entries) {
    if (entries[0].isIntersecting) {
      sticky.classList.add('sh-active');
    } else {
      sticky.classList.remove('sh-active');
      shCards.forEach(function(c) { c.classList.remove('visible'); });
      cta.classList.remove('visible');
    }
  }, { threshold: 0 });
  io.observe(section);

  // ── Scroll → frame mapping ───────────────────────────────
  var rafPending = false;
  var scrollProg = 0;

  function onScroll() {
    var rect     = section.getBoundingClientRect();
    var sectionH = section.offsetHeight - window.innerHeight;
    // -rect.top gives how many pixels the top of the section has scrolled above viewport top
    scrollProg = Math.max(0, Math.min(1, -rect.top / sectionH));
    if (!rafPending) { rafPending = true; requestAnimationFrame(render); }
  }

  function render() {
    rafPending = false;
    var p        = scrollProg;
    var frameIdx = Math.min(FRAME_COUNT - 1, Math.floor(p * FRAME_COUNT));

    if (frameIdx !== currentFrame) {
      currentFrame = frameIdx;
      drawFrame(currentFrame);
    }

    // Show/hide annotation cards based on scroll progress
    shCards.forEach(function(card) {
      var show = parseFloat(card.dataset.show);
      var hide = parseFloat(card.dataset.hide);
      if (p >= show && p < hide) card.classList.add('visible');
      else card.classList.remove('visible');
    });

    // Show CTA near end of animation
    if (p >= 0.92) cta.classList.add('visible');
    else           cta.classList.remove('visible');
  }

  // Attach scroll listener immediately — drawFrame safely no-ops until frames load
  window.addEventListener('scroll', onScroll, { passive: true });

  // ── Frame preload ────────────────────────────────────────
  // Adjust the path below to match where your frames are relative to the HTML file
  for (var i = 0; i < FRAME_COUNT; i++) {
    (function(idx) {
      var img = new Image();
      img.src = 'demo/frames/frame_' + String(idx + 1).padStart(4, '0') + '.jpg';
      img.onload = function() {
        frames[idx] = img;
        loadedCount++;
        if (loadedCount === 1) drawFrame(0);  // show frame 1 the instant it arrives
      };
      img.onerror = function() { loadedCount++; };
    })(i);
  }

})();
</script>
```

---

## Part 5 — Known Failure Modes & Fixes

This is the most important section. These are all the bugs that were encountered building this.

### 5.1 `position:sticky` silently does nothing

**Symptom:** The canvas panel scrolls off-screen. The animation "lifts a bit and then disappears." Only 1–2 frames are visible.

**Cause:** Any ancestor element with `overflow: hidden`, `overflow-x: hidden`, or `overflow: auto` breaks `sticky` in Chromium. Also broken by: Tailwind CSS CDN preflight resetting `overflow`, AOS library, `content-visibility: auto`. You only need ONE of these to be present anywhere in the ancestor chain.

**Fix:** Use `position: fixed` instead of `position: sticky`. Show/hide using `IntersectionObserver` watching the spacer div. This is the approach used in this codebase.

---

### 5.2 Snap-to transition causes `body.overflow='hidden'` on wrong element

**Symptom:** Snap feature that locks scroll momentarily at card timestamps — `window.scrollTo({ behavior: 'smooth' })` fires and then the page jumps, or gets stuck, or the snap locks and never releases.

**Cause:** The page's actual scrollable element might not be `body` or `window`. Setting `document.body.style.overflow = 'hidden'` doesn't lock the viewport if the scroll container is `<html>` or something else. Combined with `scroll-behavior: smooth` in CSS, `scrollTo()` fires badly.

**Fix:** Remove snap-stop entirely unless you specifically need it. The scroll animation works perfectly without it. If you re-add it, target the actual scroll container, not `body`.

---

### 5.3 Only 1–23 out of 96 frames animate

**Symptom:** Animation barely moves. Only a few frames change. Scrolling all the way through shows almost no motion.

**Cause (A):** The scroll event listener was attached *before* frames had loaded. The `scrollProg` variable was being set, but `frames[idx]` was `undefined` for all indices above what had loaded so far. `drawFrame()` silently returns on undefined images, so only frames that happened to be loaded at scroll time would draw.

**Fix:** `drawFrame()` no-ops if `frames[index]` is falsy. Attach the scroll listener immediately. The animation will be "choppy" until more frames load, but that's acceptable — it self-corrects as frames arrive. Show frame 0 the instant the first image loads (`if (loadedCount === 1) drawFrame(0)`).

**Cause (B):** `html { scroll-behavior: smooth }` in the CSS caused the browser to throttle/batch scroll events, so the event fired only a few times across the entire 600vh section.

**Fix:** Either remove `scroll-behavior: smooth` from `html {}` (use it on anchor links directly instead), or use an IntersectionObserver to override it with `document.documentElement.style.scrollBehavior = 'auto'` while in the section.

---

### 5.4 `currentFrame` ReferenceError / TDZ crash

**Symptom:** Page fails with `Cannot access 'currentFrame' before initialization` in the console.

**Cause:** `let currentFrame` was declared *after* `resizeCanvas()` was defined. `resizeCanvas()` calls `drawFrame(currentFrame)`. If `resizeCanvas()` is called anywhere before the `let` declaration, the Temporal Dead Zone throws.

**Fix:** Always declare `var currentFrame = 0` (or `let currentFrame = 0`) **before** defining `resizeCanvas()`. In this codebase, `var` is used to avoid the TDZ entirely.

---

### 5.5 Canvas is tiny / white space below it

**Symptom:** Canvas appears as a small box with empty space around/below it. White area visible under the canvas.

**Cause:** CSS had set `max-width: 1280px` and `margin: auto`, turning the canvas into a centered box. The sticky container was not filling the viewport.

**Fix:**
```css
#subhero-sticky { position: fixed; top:0; left:0; width:100%; height:100vh; overflow:hidden; }
#subhero-canvas { position:absolute; top:0; left:0; width:100%; height:100%; display:block; }
```
The canvas must be `position: absolute` with `inset: 0` inside the fixed container for true full-bleed.

---

### 5.6 Card text is microscopic

**Symptom:** Cards appear but text inside is 7–10px and illegible.

**Cause:** Site uses `html { font-size: 62.5% }` (common reset that makes `1rem = 10px`). Cards were sized with `.sh-title { font-size: 1rem }` etc., which renders at 10px.

**Fix:** Use `px` units directly for all card text. Card text should be:
- Label/num: `11px`
- Title: `18px`
- Description: `13px`
- Stat number: `36px`
- Stat label: `12px`

---

### 5.7 Panel entrance/exit is instant (no fade)

**Symptom:** The animation panel pops in and out with no transition.

**Cause:** Toggling only `visibility: hidden/visible` — `visibility` is a binary property and cannot be animated.

**Fix:** Use the **delayed-visibility trick**:
```css
/* Default (hidden): opacity fades to 0, THEN visibility:hidden fires 0.65s later */
transition: opacity 0.65s ease, visibility 0s linear 0.65s;
opacity: 0;
visibility: hidden;

/* Active (shown): visibility:visible fires immediately (0s delay), THEN opacity fades in */
.sh-active {
  opacity: 1;
  visibility: visible;
  transition: opacity 0.65s ease, visibility 0s linear 0s;
}
```
The `visibility 0s linear Xs` delay on the default state means the element is still technically "visible to events" while fading out — `pointer-events: none` handles that.

---

### 5.8 Frames 404 on local dev

**Symptom:** All frame images fail to load. Console shows `404 demo/frames/frame_0001.jpg`.

**Cause:** The PHP server is not running, or it was started from the wrong directory, or the router.php file is missing.

**Fix:**
```powershell
# Must be run from the workspace root (where index.html lives)
cd "C:\Users\SANDILE\Documents\Omegatek Website"
php -S 127.0.0.1:8080 router.php
```
Then open `http://127.0.0.1:8080` — do NOT open `index.html` directly in the browser (`file://`).

---

## Part 6 — Adjusting the Animation

| What you want to change | How to do it |
|---|---|
| Animation lasts longer | Increase `#subhero-section { height: 600vh }` to `800vh` or `1000vh` |
| Animation lasts shorter | Decrease height to `400vh` or `300vh` |
| Card appears earlier | Lower `data-show` value (e.g. `0.10` → `0.05`) |
| Card appears later | Raise `data-show` value |
| Card stays on screen longer | Increase gap between `data-show` and `data-hide` |
| CTA appears earlier | Change `p >= 0.92` threshold in `render()` to e.g. `p >= 0.80` |
| Fade duration | Change `0.65s` in the `#subhero-sticky` transition |
| More/fewer frames | Re-run FFmpeg with different `-vf "fps=X"`, update `FRAME_COUNT` in JS |
| Different frame folder path | Update `img.src = '...'` line in the preload loop |

---

## Part 7 — LLM Prompt Template

If you need to hand this off to a different AI assistant in the future, use this prompt:

---

> **PROMPT:**
>
> I need you to implement a scroll-driven frame animation on a webpage. Here is exactly how it works and the exact code to use. Do not deviate from this pattern — it was arrived at after extensive debugging.
>
> **The concept:** A folder of sequential JPEG frames (extracted from a video with FFmpeg at ~12fps) is preloaded in JS. A tall spacer div (600vh) acts as the scroll track. A full-screen canvas is fixed to the viewport and hidden by default. When the spacer scrolls into view, the canvas fades in. As the user scrolls through the spacer, the canvas draws frames in sequence. When the spacer leaves view, the canvas fades out.
>
> **CRITICAL rules — do not break these:**
>
> 1. **Use `position: fixed` on the canvas panel, NOT `position: sticky`.** Sticky breaks silently in any page with `overflow-x: hidden` on html/body, Tailwind CDN, AOS, or `content-visibility: auto`. Fixed is immune to all of these.
>
> 2. **Show/hide the fixed panel using IntersectionObserver on the spacer div.** Toggle a `.sh-active` class. Use the delayed-visibility CSS trick for smooth fades (see CSS below).
>
> 3. **Declare `var currentFrame = 0` BEFORE defining `resizeCanvas()`.** `resizeCanvas()` calls `drawFrame(currentFrame)` — if `currentFrame` is in the TDZ, the page crashes.
>
> 4. **Attach the scroll listener immediately, not after frames load.** `drawFrame()` no-ops if the image isn't loaded yet. Show frame 0 when `loadedCount === 1` (first image arrives).
>
> 5. **Use `px` units for ALL annotation card text, not `rem`.** This site uses `html { font-size: 62.5% }` so `1rem = 10px`. Card titles, descriptions, and stats must use `18px`, `13px`, `36px` etc.
>
> 6. **Canvas fill strategy is cover-fit, not contain.** Use `Math.max(canvasW/iw, canvasH/ih)` as the scale factor so the video fills the viewport edge-to-edge.
>
> 7. **Do not implement scroll snap / snap-stop.** The version with snap-stop used `body.overflow = 'hidden'` which conflicts with the scroll container and breaks the animation.
>
> **File locations in this project:**
> - Frames: `demo/frames/frame_0001.jpg` … `frame_0096.jpg` (relative to site root `index.html`)
> - CSS: add to `css/style.css`
> - JS: add as `<script>` block at bottom of `index.html` before `</body>`
> - HTML section: insert after `<section class="hero-modern">` closes
>
> **The working CSS and JS are below. Implement these exactly.**
>
> [Paste the full CSS block from Part 3 and the full JS block from Part 4 of this document]

---

## Part 8 — File Inventory

| File | Purpose |
|---|---|
| `demo/frames/frame_0001.jpg` … `frame_0096.jpg` | 96 JPEG frames extracted from the video |
| `images2/Phone_deconstruction_animation.mp4` | Original source video (1280×720, 24fps, 8s) |
| `css/style.css` (lines ~342–510) | All animation CSS |
| `index.html` (lines ~576–622) | HTML for `#subhero-section` |
| `index.html` (lines ~1947–2055) | JavaScript IIFE for the animation |
| `router.php` | PHP dev server router (handles extensionless URLs) |
| `demo/index.html` | Standalone demo page (isolated, always works as reference) |

---

## Part 9 — Quick Checklist Before Going Live

- [ ] PHP server is running from the workspace root (`php -S 127.0.0.1:8080 router.php`)
- [ ] `(Get-ChildItem demo\frames\frame_*.jpg).Count` returns `96`
- [ ] Open browser console — no 404s for frame images
- [ ] Hard refresh (`Ctrl+Shift+R`) after any CSS change to clear browser cache
- [ ] Scroll past the hero — canvas fades in smoothly
- [ ] Scroll through — all 4 annotation cards appear and disappear
- [ ] Scroll to end — CTA button fades in
- [ ] Scroll past section — canvas fades out smoothly
- [ ] Test on mobile: cards are still legible, animation plays

---

*Last updated: March 2026 — Omegatek Solutions homepage.*
