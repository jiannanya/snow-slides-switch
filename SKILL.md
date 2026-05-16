---
name: snow-slides-switch
description: 'HTML slide page-transition animation system. Implements multiple slide-switching effects (slide-left, slide-right, slide-up, slide-down, zoom, flip, fade, blur, elastic-scale) for single-page HTML presentations, while coordinating per-element entrance animations so that the slide transition and element animations play simultaneously — never sequentially — eliminating the double-animation problem. Use for: HTML PPT/Slides authoring, single-page presentation animation systems, smooth slide transitions that do not break element entrance animations. Keywords: slides switch transition animation ppt stagger entrance overlap.'
argument-hint: 'Describe the target presentation style or desired transition effect'
---

# snow-slides-switch — HTML Slide Transition Animation System

## Core Design Principle

> **No Double-Animation**: The slide transition (page movement) and element entrance animations run **simultaneously in parallel**, not one after the other. Element animations begin 50 ms into the slide transition, so both complete within the same 600 ms window.

```
Timeline:
  0ms  ─── new slide starts moving in from off-screen  (CSS transition)
 50ms  ───── per-element entrance animations begin      (JS applyAnimations)
600ms  ─────────── slide transition ends, page fully visible
650ms  ──────────────── cleanup  (remove trans-* class, hide old slide)
```

---

## I. Complete CSS

### 1. Stage Layout

```css
/* Stage container: needs position + perspective for 3-D flip */
#app {
  position: relative;
  width: var(--slide-w);   /* e.g. 1280px */
  height: var(--slide-h);  /* e.g.  720px */
  perspective: 2000px;
}

/* Base slide style */
.slide {
  position: absolute;
  inset: 0;
  display: none;
  /* Core: CSS transition drives every switching effect */
  transition:
    transform 0.6s cubic-bezier(.4, 0, .2, 1),
    opacity   0.6s ease,
    filter    0.6s ease;
}

/* Currently active slide */
.slide.active {
  display: flex; /* or block — adapt to your layout */
}
```

### 2. Off-Screen States (starting position before a slide enters)

```css
/* Horizontal / Vertical slides */
.slide.trans-left  { transform: translateX(-100%); opacity: 0; }
.slide.trans-right { transform: translateX(100%);  opacity: 0; }
.slide.trans-up    { transform: translateY(-100%); opacity: 0; }
.slide.trans-down  { transform: translateY(100%);  opacity: 0; }

/* Zoom */
.slide.trans-zoom    { transform: scale(.7); opacity: 0; }
.slide.trans-scale-e { transform: scale(.3); opacity: 0; } /* elastic zoom */

/* 3-D Flip */
.slide.trans-flip { transform: rotateY(90deg); opacity: 0; }

/* Fade */
.slide.trans-fade { opacity: 0; }

/* Blur-fade */
.slide.trans-blur { opacity: 0; filter: blur(12px); }
```

### 3. Active States (adding `.active` triggers the CSS transition into view)

```css
/* Slide types — all return to origin */
.slide.trans-left.active,
.slide.trans-right.active,
.slide.trans-up.active,
.slide.trans-down.active {
  transform: translate(0);
  opacity: 1;
}

/* Zoom types */
.slide.trans-zoom.active    { transform: scale(1); opacity: 1; }
.slide.trans-scale-e.active { transform: scale(1); opacity: 1; }

/* Flip */
.slide.trans-flip.active { transform: rotateY(0); opacity: 1; }

/* Fade */
.slide.trans-fade.active { opacity: 1; }

/* Blur-fade */
.slide.trans-blur.active { opacity: 1; filter: blur(0); }
```

### 4. Element Entrance Keyframes

```css
/* Primary entrance — subtle upward lift + scale fade-in */
@keyframes fadeInStagger {
  from { opacity: 0; transform: translateY(30px) scale(.95); }
  to   { opacity: 1; transform: translateY(0)    scale(1);   }
}

/* Optional extras */
@keyframes fadeInUp    { from { opacity:0; transform:translateY(40px)  } to { opacity:1; transform:translateY(0) } }
@keyframes fadeInLeft  { from { opacity:0; transform:translateX(-50px) } to { opacity:1; transform:translateX(0) } }
@keyframes fadeInRight { from { opacity:0; transform:translateX(50px)  } to { opacity:1; transform:translateX(0) } }
@keyframes scaleIn     { from { opacity:0; transform:scale(.5)  } to { opacity:1; transform:scale(1) } }
@keyframes blurIn      { from { opacity:0; filter:blur(12px)    } to { opacity:1; filter:blur(0)    } }
@keyframes zoomInUp    { from { opacity:0; transform:scale(.6) translateY(40px) } to { opacity:1; transform:scale(1) translateY(0) } }
```

### 5. Continuous Animation Classes (must be reset on each slide enter)

```css
/* Examples: pulse, float — reset and restarted by JS on slide enter */
@keyframes pulse { 0%,100%{transform:scale(1)} 50%{transform:scale(1.06)} }
@keyframes float { 0%,100%{transform:translateY(0)} 50%{transform:translateY(-12px)} }

.pulse      { animation: pulse 2.5s ease-in-out infinite; }
.float      { animation: float 3s   ease-in-out infinite; }
.float-slow { animation: float 5s   ease-in-out infinite; }
```

---

## II. Complete JavaScript

```javascript
/* ========================================================
   snow-slides-switch — Slide Transition Engine
   ======================================================== */

let current = 0;
let transitioning = false;

const slides = document.querySelectorAll('.slide');
const total  = slides.length;

// Transition pool: cycled through during sequential navigation
const TRANSITIONS = [
  'trans-left', 'trans-right', 'trans-up', 'trans-down',
  'trans-zoom', 'trans-fade', 'trans-blur', 'trans-scale-e'
];

// ⚠️ All continuous-animation class selectors that must be reset on enter
const CONTINUOUS_ANIM_CLASSES = [
  '.pulse', '.pulse-slow', '.pulse-glow', '.pulse-glow-slow',
  '.float', '.float-slow', '.float-rotate',
  '.heartbeat', '.heartbeat-slow',
  '.breathe', '.subtle-glow', '.gentle-swing',
  '.color-shift', '.border-pulse', '.scale-breath',
  '.shimmer-text', '.shimmer-slow',
  '.bg-animate', '.bg-animate-fast'
].join(',');

/**
 * Navigate to a slide.
 * @param {number} n    Target slide index
 * @param {number} dir  Direction: 1 = forward, -1 = backward, 0 = direct jump
 */
function showSlide(n, dir) {
  if (transitioning) return;
  transitioning = true;

  const old = slides[current];
  const idx = (n + total) % total;
  const nu  = slides[idx];

  if (idx === current) { transitioning = false; return; }

  // 1. Deactivate current slide (removes .active → old slide fades/slides out)
  old.classList.remove('active');

  // 2. Choose transition class
  let transClass;
  if (dir === 1)       transClass = TRANSITIONS[idx % TRANSITIONS.length];
  else if (dir === -1) transClass = TRANSITIONS[(total - idx) % TRANSITIONS.length];
  else                 transClass = nu.getAttribute('data-trans') || 'trans-fade';
  transClass = 'trans-' + transClass.replace(/^trans-/, ''); // normalise prefix

  // 3. Position new slide off-screen and make it participate in layout
  nu.classList.add(transClass);
  nu.style.display = 'flex';

  // 4. Double rAF: let browser commit the off-screen state before starting transition
  requestAnimationFrame(() => {
    requestAnimationFrame(() => {
      // 5. Add .active → CSS transition animates slide into view
      nu.classList.add('active');

      // ⚠️ KEY: fire element entrance animations 50ms INTO the slide transition
      // Overlapping playback prevents a double-animation pause after the slide lands
      setTimeout(() => applyAnimations(nu), 50);

      // 6. Clean up after transition ends (600ms CSS duration + 50ms buffer)
      setTimeout(() => {
        nu.classList.remove(transClass);
        old.style.display = 'none';
        current = idx;
        updateUI(); // update progress bar, slide counter, etc. (implement yourself)
        transitioning = false;
      }, 650);
    });
  });
}

/**
 * Reset and replay all animations on the entering slide.
 * Pattern: set animation='none', force reflow, then reassign to restart from zero.
 */
function applyAnimations(slide) {
  // — Staggered entrance elements (.anim-stagger + data-d delay attribute)
  const staggerEls = slide.querySelectorAll('.anim-stagger');
  staggerEls.forEach((el, i) => {
    el.style.animation = 'none';
    el.offsetHeight; // force reflow — clears previous animation state
    const d     = parseFloat(el.getAttribute('data-d') || 0);
    const delay = d * 500 + i * 45; // ms; tune as needed
    el.style.animation = `fadeInStagger .5s cubic-bezier(.25,.46,.45,.94) ${delay}ms both`;
    //                                                                      ↑ 'both' keeps element
    //                                                                        in 'from' state before
    //                                                                        animation fires (no flash)
  });

  // — Reset continuous animations so they restart from the beginning on entry
  slide.querySelectorAll(CONTINUOUS_ANIM_CLASSES).forEach(el => {
    el.style.animation = 'none';
    el.offsetHeight;
    el.style.animation = ''; // empty string (not 'none') restores the CSS class animation
  });
}

// ——— Navigation helpers ———
function nextSlide() { showSlide(current + 1,  1); }
function prevSlide() { showSlide(current - 1, -1); }

// Keyboard & mouse-wheel navigation
document.addEventListener('keydown', e => {
  if (e.key === 'ArrowRight' || e.key === 'ArrowDown' || e.key === ' ')
    { e.preventDefault(); nextSlide(); }
  if (e.key === 'ArrowLeft' || e.key === 'ArrowUp')
    { e.preventDefault(); prevSlide(); }
  if (e.key === 'Home') { e.preventDefault(); showSlide(0, 0); }
  if (e.key === 'End')  { e.preventDefault(); showSlide(total - 1, 0); }
});

document.addEventListener('wheel', e => {
  if (e.deltaY > 30)  { e.preventDefault(); nextSlide(); }
  if (e.deltaY < -30) { e.preventDefault(); prevSlide(); }
}, { passive: false });

// Touch swipe
let touchStartY = 0;
document.addEventListener('touchstart', e => { touchStartY = e.touches[0].clientY; });
document.addEventListener('touchend',   e => {
  const diff = touchStartY - e.changedTouches[0].clientY;
  if (Math.abs(diff) > 50) { diff > 0 ? nextSlide() : prevSlide(); }
});

// Initialise first slide
window.addEventListener('load', () => {
  setTimeout(() => applyAnimations(slides[0]), 100);
});
```

---

## III. HTML Usage

### Slide Markup Template

```html
<!-- data-trans: transition used when jumping directly to this slide (dir=0) -->
<div class="slide active" data-slide="0" data-trans="blur">
  <!-- Add .anim-stagger + data-d (delay in seconds) for staggered entrance -->
  <h1  class="anim-stagger" data-d="0">Title</h1>
  <p   class="anim-stagger" data-d="0.15">Subtitle — appears 150 ms later</p>
  <div class="anim-stagger" data-d="0.3">Content block — appears 300 ms later</div>
</div>

<div class="slide" data-slide="1" data-trans="right">
  <h2 class="anim-stagger" data-d="0">Slide 2 heading</h2>
</div>

<div class="slide" data-slide="2" data-trans="zoom">
  <!-- Continuous-animation classes are auto-reset on slide enter -->
  <div class="anim-stagger float" data-d="0">🚀</div>
</div>
```

### `data-trans` Values

| Value      | Effect              | Transform                          |
|------------|---------------------|------------------------------------|
| `left`     | Slide in from left  | `translateX(-100%) → 0`            |
| `right`    | Slide in from right | `translateX(100%) → 0`             |
| `up`       | Slide in from top   | `translateY(-100%) → 0`            |
| `down`     | Slide in from bottom| `translateY(100%) → 0`             |
| `zoom`     | Zoom fade-in        | `scale(0.7) → scale(1)`            |
| `scale-e`  | Elastic zoom        | `scale(0.3) → scale(1)`            |
| `flip`     | 3-D Y-axis flip     | `rotateY(90deg) → rotateY(0)`      |
| `fade`     | Pure fade           | `opacity 0 → 1`                    |
| `blur`     | Blur fade-in        | `blur(12px)+opacity 0 → normal`    |

### `data-d` Delay Reference

```
data-d="0"    → immediate (within 50 ms of transition start)
data-d="0.1"  → ~100 ms delay
data-d="0.3"  → ~200 ms delay  (typical body text)
data-d="0.5"  → ~300 ms delay  (decorative / trailing elements)
```

Formula: `actual delay (ms) = data-d × 500 + element index × 45`

---

## IV. No-Double-Animation: Deep Dive

### The Problem

If element entrance animations fire **after** the slide transition ends, users see:
1. Slide animates in (600 ms)
2. Brief visual pause
3. Elements animate in again

This two-phase motion feels disjointed and amateurish.

### The Solution: Temporal Overlap

```
showSlide() call sequence:

  requestAnimationFrame × 2
    └── nu.classList.add('active')          ← slide transition starts
        setTimeout(applyAnimations, 50)     ← elements start 50ms later (OVERLAPPING)
        setTimeout(cleanup, 650)            ← cleanup after transition

[CSS transition:    600ms]  ████████████████████████████████
[Elements offset:    50ms]       ░░░░░░░░░░░░░░░░░░░░░░░░░░░░
                            ↑50ms
```

Key points:
- `animation-fill-mode: both` keeps elements in their `from` (invisible) state before the animation fires — no content flash
- Element animations (500 ms) plus maximum stagger (~350 ms) finish comfortably within the 650 ms window
- `el.style.animation = 'none'` + `el.offsetHeight` (forced reflow) guarantees every animation restarts from frame zero regardless of prior state

### Resetting Continuous Animations Correctly

```javascript
// ✅ Correct: empty string (not 'none') restores the CSS class animation
el.style.animation = 'none';
el.offsetHeight; // force reflow
el.style.animation = ''; // ← empty, NOT 'none'

// ❌ Wrong: skipping the reflow; browser may not reset
el.style.animation = 'none';
el.style.animation = '';
```

---

## V. Quick Integration Checklist

- [ ] Copy **CSS layout** + **transition classes** + **keyframes** into `<style>`
- [ ] Copy **JavaScript engine** into `<script>` (implement `updateUI()` as needed)
- [ ] Add `data-trans` attribute to every `.slide` element
- [ ] Add `.anim-stagger` + `data-d` to all elements that need entrance animation
- [ ] First slide: `class="slide active"` — remaining slides: `class="slide"`
- [ ] Confirm `CONTINUOUS_ANIM_CLASSES` lists every continuous-animation class used in the project
