# snow-slides-switch

> 🌐 Language: **English** | [中文](./README.zh.md)

A lightweight, zero-dependency **HTML slide transition animation system** that coordinates per-element entrance animations with page-switching effects — so users always see one smooth, cohesive motion instead of two sequential animations.

---

## ✨ Features

- **9 built-in transition effects** — slide (left / right / up / down), zoom, elastic-zoom, 3-D flip, fade, blur-fade
- **No double-animation** — element entrances overlap the slide transition; they never play a second time after the page lands
- **CSS-driven transitions** — all movement is handled by `transform` + `opacity` + `filter` transitions; no JS animation loop
- **Per-element stagger** — `data-d` attribute controls each element's entrance delay independently
- **Continuous animation reset** — `pulse`, `float`, and other looping animations automatically restart when a slide enters
- **Full navigation support** — arrow keys, spacebar, mouse wheel, touch swipe
- **Zero dependencies** — pure HTML + CSS + vanilla JS; works in any browser that supports CSS transitions

---

## 🚀 Quick Start

### 1. Add the CSS

```html
<style>
/* Stage */
#app {
  position: relative;
  width: 1280px; height: 720px;
  perspective: 2000px;
}

/* Slide base */
.slide {
  position: absolute; inset: 0; display: none;
  transition: transform .6s cubic-bezier(.4,0,.2,1), opacity .6s ease, filter .6s ease;
}
.slide.active { display: flex; }

/* Off-screen states */
.slide.trans-left   { transform: translateX(-100%); opacity: 0; }
.slide.trans-right  { transform: translateX(100%);  opacity: 0; }
.slide.trans-up     { transform: translateY(-100%); opacity: 0; }
.slide.trans-down   { transform: translateY(100%);  opacity: 0; }
.slide.trans-zoom   { transform: scale(.7);         opacity: 0; }
.slide.trans-scale-e{ transform: scale(.3);         opacity: 0; }
.slide.trans-flip   { transform: rotateY(90deg);    opacity: 0; }
.slide.trans-fade   { opacity: 0; }
.slide.trans-blur   { opacity: 0; filter: blur(12px); }

/* Active (on-screen) states */
.slide.trans-left.active,  .slide.trans-right.active,
.slide.trans-up.active,    .slide.trans-down.active  { transform: translate(0); opacity: 1; }
.slide.trans-zoom.active,  .slide.trans-scale-e.active { transform: scale(1);   opacity: 1; }
.slide.trans-flip.active   { transform: rotateY(0); opacity: 1; }
.slide.trans-fade.active   { opacity: 1; }
.slide.trans-blur.active   { opacity: 1; filter: blur(0); }

/* Element entrance keyframe */
@keyframes fadeInStagger {
  from { opacity: 0; transform: translateY(30px) scale(.95); }
  to   { opacity: 1; transform: translateY(0)    scale(1);   }
}
</style>
```

### 2. Add the HTML

```html
<div id="app">

  <!-- First slide: class="slide active", data-trans = entrance effect -->
  <div class="slide active" data-slide="0" data-trans="blur">
    <h1  class="anim-stagger" data-d="0"   >Hello, World!</h1>
    <p   class="anim-stagger" data-d="0.15">Subtitle appears 150 ms later</p>
    <div class="anim-stagger" data-d="0.3" >Body content at 300 ms</div>
  </div>

  <!-- Subsequent slides: class="slide" only -->
  <div class="slide" data-slide="1" data-trans="right">
    <h2 class="anim-stagger" data-d="0">Slide Two</h2>
  </div>

  <div class="slide" data-slide="2" data-trans="zoom">
    <div class="anim-stagger float" data-d="0">🚀</div>
  </div>

</div>
```

### 3. Add the JavaScript

```html
<script>
let current = 0, transitioning = false;
const slides = document.querySelectorAll('.slide');
const total  = slides.length;
const TRANSITIONS = ['trans-left','trans-right','trans-up','trans-down',
                     'trans-zoom','trans-fade','trans-blur','trans-scale-e'];
const CONTINUOUS = '.pulse,.float,.float-slow,.heartbeat,.breathe'; // extend as needed

function showSlide(n, dir) {
  if (transitioning) return;
  transitioning = true;
  const old = slides[current], idx = (n + total) % total, nu = slides[idx];
  if (idx === current) { transitioning = false; return; }

  old.classList.remove('active');

  let tc;
  if (dir === 1)       tc = TRANSITIONS[idx % TRANSITIONS.length];
  else if (dir === -1) tc = TRANSITIONS[(total - idx) % TRANSITIONS.length];
  else                 tc = nu.getAttribute('data-trans') || 'trans-fade';
  tc = 'trans-' + tc.replace(/^trans-/, '');

  nu.classList.add(tc);
  nu.style.display = 'flex';

  requestAnimationFrame(() => requestAnimationFrame(() => {
    nu.classList.add('active');
    setTimeout(() => applyAnimations(nu), 50);   // overlap with transition
    setTimeout(() => {
      nu.classList.remove(tc);
      old.style.display = 'none';
      current = idx;
      transitioning = false;
    }, 650);
  }));
}

function applyAnimations(slide) {
  slide.querySelectorAll('.anim-stagger').forEach((el, i) => {
    el.style.animation = 'none'; el.offsetHeight;
    const delay = parseFloat(el.getAttribute('data-d') || 0) * 500 + i * 45;
    el.style.animation = `fadeInStagger .5s cubic-bezier(.25,.46,.45,.94) ${delay}ms both`;
  });
  slide.querySelectorAll(CONTINUOUS).forEach(el => {
    el.style.animation = 'none'; el.offsetHeight; el.style.animation = '';
  });
}

function nextSlide() { showSlide(current + 1,  1); }
function prevSlide() { showSlide(current - 1, -1); }

document.addEventListener('keydown', e => {
  if (['ArrowRight','ArrowDown',' '].includes(e.key)) { e.preventDefault(); nextSlide(); }
  if (['ArrowLeft','ArrowUp'].includes(e.key))         { e.preventDefault(); prevSlide(); }
});
document.addEventListener('wheel', e => {
  if (e.deltaY > 30)  { e.preventDefault(); nextSlide(); }
  if (e.deltaY < -30) { e.preventDefault(); prevSlide(); }
}, { passive: false });

window.addEventListener('load', () => setTimeout(() => applyAnimations(slides[0]), 100));
</script>
```

---

## 📖 Reference

### Transition Effects (`data-trans`)

| Value      | Entrance direction  | CSS transform used                   |
|------------|---------------------|--------------------------------------|
| `left`     | From left           | `translateX(-100%) → 0`              |
| `right`    | From right          | `translateX(100%) → 0`               |
| `up`       | From top            | `translateY(-100%) → 0`              |
| `down`     | From bottom         | `translateY(100%) → 0`               |
| `zoom`     | Zoom in             | `scale(0.7) → scale(1)`              |
| `scale-e`  | Elastic zoom        | `scale(0.3) → scale(1)`              |
| `flip`     | 3-D Y-axis flip     | `rotateY(90deg) → rotateY(0)`        |
| `fade`     | Fade                | `opacity 0 → 1`                      |
| `blur`     | Blur fade           | `blur(12px) + opacity 0 → normal`    |

### Element Delay (`data-d`)

Actual delay (ms) = `data-d × 500 + element_index × 45`

| `data-d` | Approximate delay |
|----------|-------------------|
| `0`      | 0–45 ms           |
| `0.1`    | ~95 ms            |
| `0.3`    | ~195 ms           |
| `0.5`    | ~295 ms           |

---

## 🧠 How It Works

### Why No Double-Animation?

The double-animation problem occurs when element entrances fire **after** the slide finishes moving in, producing a noticeable two-phase motion:

```
❌ Sequential (bad):
   [slide transition 600ms] → pause → [element animation 500ms]

✅ Overlapping (this system):
   [slide transition 600ms]  ████████████████████████████
   [element animation 500ms]      ░░░░░░░░░░░░░░░░░░░░░░░
                              ↑ 50ms offset
```

### Double `requestAnimationFrame`

```javascript
requestAnimationFrame(() =>        // frame 1: browser lays out off-screen position
  requestAnimationFrame(() => {    // frame 2: browser paints, THEN we add .active
    nu.classList.add('active');    // → triggers CSS transition from off-screen to on-screen
  })
);
```

Without this double-rAF, the browser may merge the style updates and skip the "off-screen" state, causing the transition to be invisible.

### Animation Reset Pattern

```javascript
el.style.animation = 'none';  // suspend
el.offsetHeight;               // force reflow (this is the magic)
el.style.animation = '';       // restore — browser replays from frame 0
```

The `offsetHeight` read forces the browser to perform layout, flushing the animation state. Without it, setting `animation = ''` immediately after `animation = 'none'` may have no effect.

---

## 📁 File Structure

```
snow-slides-switch/
├── index.html      # Full working example (77-slide presentation)
├── SKILL.md        # Copilot skill — Chinese
├── SKILL.en.md     # Copilot skill — English
├── README.md       # This file (English)
└── README.zh.md    # Chinese documentation
```

---

## 📄 License

MIT
