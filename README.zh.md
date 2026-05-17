# snow-slides-switch

> 🌐 语言：[English](./README.md) | **中文**

一个轻量级、零依赖的 **HTML 幻灯片切换动画系统**，将页面切换过渡效果与元素入场动画协调同步播放——用户始终看到一次流畅、连贯的运动，而不是两段顺序播放的独立动画。

---

## ✨ 特性

- **9 种内置切换效果** — 滑动（左 / 右 / 上 / 下）、缩放、弹性缩放、3D 翻转、淡入、模糊淡入
- **防二次动画** — 元素入场动画与页面切换动画重叠播放，不会在页面落定后再次重播
- **CSS 驱动过渡** — 所有运动由 `transform` + `opacity` + `filter` 的 CSS transition 处理，无 JS 动画循环
- **元素分批入场** — `data-d` 属性独立控制每个元素的入场延迟
- **持续动画重置** — `pulse`、`float` 等循环动画在页面进入时自动从头重播
- **全面导航支持** — 方向键、空格键、鼠标滚轮、触屏滑动
- **零依赖** — 纯 HTML + CSS + 原生 JS，兼容所有支持 CSS transition 的浏览器

---

## 🚀 快速开始

### 1. 添加 CSS

```html
<style>
/* 舞台容器 */
#app {
  position: relative;
  width: 1280px; height: 720px;
  perspective: 2000px;
}

/* 幻灯片基础样式 */
.slide {
  position: absolute; inset: 0; display: none;
  transition: transform .6s cubic-bezier(.4,0,.2,1), opacity .6s ease, filter .6s ease;
}
.slide.active { display: flex; }

/* 各效果的离屏初始状态 */
.slide.trans-left   { transform: translateX(-100%); opacity: 0; }
.slide.trans-right  { transform: translateX(100%);  opacity: 0; }
.slide.trans-up     { transform: translateY(-100%); opacity: 0; }
.slide.trans-down   { transform: translateY(100%);  opacity: 0; }
.slide.trans-zoom   { transform: scale(.7);         opacity: 0; }
.slide.trans-scale-e{ transform: scale(.3);         opacity: 0; }
.slide.trans-flip   { transform: rotateY(90deg);    opacity: 0; }
.slide.trans-fade   { opacity: 0; }
.slide.trans-blur   { opacity: 0; filter: blur(12px); }

/* 激活状态（触发 CSS transition 进场） */
.slide.trans-left.active,  .slide.trans-right.active,
.slide.trans-up.active,    .slide.trans-down.active  { transform: translate(0); opacity: 1; }
.slide.trans-zoom.active,  .slide.trans-scale-e.active { transform: scale(1);   opacity: 1; }
.slide.trans-flip.active   { transform: rotateY(0); opacity: 1; }
.slide.trans-fade.active   { opacity: 1; }
.slide.trans-blur.active   { opacity: 1; filter: blur(0); }

/* 元素入场关键帧 */
@keyframes fadeInStagger {
  from { opacity: 0; transform: translateY(30px) scale(.95); }
  to   { opacity: 1; transform: translateY(0)    scale(1);   }
}
</style>
```

### 2. 添加 HTML

```html
<div id="app">

  <!-- 第一页：class="slide active"，data-trans 指定进场效果 -->
  <div class="slide active" data-slide="0" data-trans="blur">
    <h1  class="anim-stagger" data-d="0"   >你好，世界！</h1>
    <p   class="anim-stagger" data-d="0.15">副标题，150ms 后出现</p>
    <div class="anim-stagger" data-d="0.3" >内容块，300ms 后出现</div>
  </div>

  <!-- 后续页：只有 class="slide" -->
  <div class="slide" data-slide="1" data-trans="right">
    <h2 class="anim-stagger" data-d="0">第二页</h2>
  </div>

  <div class="slide" data-slide="2" data-trans="zoom">
    <!-- 持续动画类（.float 等）在页面进入时自动重置重播 -->
    <div class="anim-stagger float" data-d="0">🚀</div>
  </div>

</div>
```

### 3. 添加 JavaScript

```html
<script>
let current = 0, transitioning = false;
const slides = document.querySelectorAll('.slide');
const total  = slides.length;
const TRANSITIONS = ['trans-left','trans-right','trans-up','trans-down',
                     'trans-zoom','trans-fade','trans-blur','trans-scale-e'];
const CONTINUOUS = '.pulse,.float,.float-slow,.heartbeat,.breathe'; // 按需扩展

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
    setTimeout(() => applyAnimations(nu), 50);   // 与切换动画重叠
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

## 📖 参考手册

### 切换效果（`data-trans`）

| 值         | 进场方向         | CSS 变换                              |
|------------|------------------|---------------------------------------|
| `left`     | 从左侧滑入       | `translateX(-100%) → 0`               |
| `right`    | 从右侧滑入       | `translateX(100%) → 0`                |
| `up`       | 从顶部滑入       | `translateY(-100%) → 0`               |
| `down`     | 从底部滑入       | `translateY(100%) → 0`                |
| `zoom`     | 缩小淡入         | `scale(0.7) → scale(1)`               |
| `scale-e`  | 弹性缩放淡入     | `scale(0.3) → scale(1)`               |
| `flip`     | Y 轴 3D 翻转     | `rotateY(90deg) → rotateY(0)`         |
| `fade`     | 纯淡入           | `opacity 0 → 1`                       |
| `blur`     | 模糊淡入         | `blur(12px) + opacity 0 → 正常`       |

### 元素延迟（`data-d`）

实际延迟（ms）= `data-d × 500 + 元素序号 × 45`

| `data-d` | 大约延迟   |
|----------|------------|
| `0`      | 0–45 ms    |
| `0.1`    | 约 95 ms   |
| `0.3`    | 约 195 ms  |
| `0.5`    | 约 295 ms  |

---

## 🧠 工作原理

### 为何不会产生二次动画？

二次动画问题出现在元素入场动画于页面切换**结束之后**才开始播放，导致用户明显感知到两段独立运动：

```
❌ 顺序播放（不好）：
   [页面切换 600ms] → 停顿 → [元素入场 500ms]

✅ 重叠播放（本系统）：
   [页面切换 600ms]  ████████████████████████████
   [元素入场 500ms]       ░░░░░░░░░░░░░░░░░░░░░░░
                     ↑ 50ms 偏移起点
```

### 双层 `requestAnimationFrame`

```javascript
requestAnimationFrame(() =>       // 第 1 帧：浏览器布局离屏位置
  requestAnimationFrame(() => {   // 第 2 帧：浏览器绘制完成后再添加 .active
    nu.classList.add('active');   // → 触发 CSS transition 从离屏到可见
  })
);
```

缺少这个双层 rAF，浏览器可能合并样式更新，跳过"离屏状态"，导致过渡不可见。

### 动画重置范式

```javascript
el.style.animation = 'none';  // 暂停动画
el.offsetHeight;               // 强制回流（关键！）
el.style.animation = '';       // 恢复 —— 浏览器从第 0 帧重播
```

读取 `offsetHeight` 强制浏览器执行布局，清空动画状态。若不执行此步骤，紧接着将 `animation = ''` 可能无效。

---

## 🎬 在线演示

用任意现代浏览器直接打开 [`assets/demo.html`](./assets/demo.html)，无需服务器。

演示文稿共 **78 页**，主题为《Harness Engineering · 驾驭工程》，完整展示本系统所有特性：
- 全部 9 种切换效果（各章节封面各不相同）
- 所有内容页的元素分批入场动画
- 持续动画：`pulse`、`float`、`shimmer-text` 等
- ECharts 图表与页面切换协调
- 英雄页粒子系统、点击涟漪效果
- 键盘、鼠标滚轮、触屏导航

---

## 📁 文件结构

```
snow-slides-switch/
├── assets/
│   └── demo.html       # 内置演示文稿 —— 78 页 PPT（浏览器直接打开）
├── SKILL.md            # Copilot 技能文件 — 英文
├── SKILL_cn.md         # Copilot 技能文件 — 中文
├── README.md           # 英文文档
└── README.zh.md        # 本文件（中文）
```

---

## 📄 许可证

MIT
