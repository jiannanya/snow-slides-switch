---
name: snow-slides-switch
description: 'HTML幻灯片页面切换动画系统。为单页HTML演示文稿（PPT）实现多种页面切换过渡效果（左移、右移、上移、下移、缩放、翻转、淡入、模糊、弹性缩放），同时协调页面元素的入场动画，确保切换动画与元素动画同步叠加播放，不产生二次重复动画。适用场景：HTML PPT/Slides制作、单页幻灯片动画系统搭建、需要平滑页面切换且不破坏元素入场动画的场景。关键词：slides switch transition animation ppt 幻灯片切换 过渡动画 入场动画 stagger。'
argument-hint: '描述目标PPT风格或需要的切换效果'
---

# snow-slides-switch — HTML幻灯片切换动画系统

## 核心设计原则

> **防二次动画**：页面切换动画（slide transition）与页面元素入场动画（element entrance）**同步重叠播放**，而非顺序播放。切换动画开始后 50ms 即触发元素动画，两者在 600ms 切换窗口内并行完成。

```
时间轴:
0ms    ─── 新页面从屏幕外开始滑入（CSS transition）
50ms   ───── 页面元素同步开始入场动画（JS applyAnimations）
600ms  ─────────── 滑入动画结束，页面已完全可见
650ms  ──────────────── 清理工作（移除 trans-* 类、隐藏旧页面）
```

---

## 一、CSS 完整代码

### 1. 基础布局

```css
/* 舞台容器：须有相对定位和 perspective 以支持 3D 翻转 */
#app {
  position: relative;
  width: var(--slide-w);   /* 例: 1280px */
  height: var(--slide-h);  /* 例: 720px */
  perspective: 2000px;
}

/* 幻灯片基础样式 */
.slide {
  position: absolute;
  inset: 0;
  display: none;
  /* 核心：CSS transition 驱动所有切换动画 */
  transition:
    transform 0.6s cubic-bezier(.4, 0, .2, 1),
    opacity   0.6s ease,
    filter    0.6s ease;
}

/* 当前激活的页面 */
.slide.active {
  display: flex; /* 或 block，按内容布局需求设定 */
}
```

### 2. 各切换效果的"离屏状态"（进场前的初始位置）

```css
/* 水平 / 垂直 滑动 */
.slide.trans-left  { transform: translateX(-100%); opacity: 0; }
.slide.trans-right { transform: translateX(100%);  opacity: 0; }
.slide.trans-up    { transform: translateY(-100%); opacity: 0; }
.slide.trans-down  { transform: translateY(100%);  opacity: 0; }

/* 缩放 */
.slide.trans-zoom    { transform: scale(.7);         opacity: 0; }
.slide.trans-scale-e { transform: scale(.3);         opacity: 0; } /* 弹性缩放 */

/* 3D 翻转 */
.slide.trans-flip { transform: rotateY(90deg); opacity: 0; }

/* 淡入 */
.slide.trans-fade { opacity: 0; }

/* 模糊淡入 */
.slide.trans-blur { opacity: 0; filter: blur(12px); }
```

### 3. 激活状态（`.active` 触发 CSS transition 完成进场）

```css
/* 滑动类：统一归位 */
.slide.trans-left.active,
.slide.trans-right.active,
.slide.trans-up.active,
.slide.trans-down.active {
  transform: translate(0);
  opacity: 1;
}

/* 缩放类 */
.slide.trans-zoom.active    { transform: scale(1); opacity: 1; }
.slide.trans-scale-e.active { transform: scale(1); opacity: 1; }

/* 翻转 */
.slide.trans-flip.active { transform: rotateY(0); opacity: 1; }

/* 淡入 */
.slide.trans-fade.active { opacity: 1; }

/* 模糊淡入 */
.slide.trans-blur.active { opacity: 1; filter: blur(0); }
```

### 4. 元素入场动画关键帧

```css
/* 元素入场：带轻微上移 + 缩放的淡入，适合大多数内容块 */
@keyframes fadeInStagger {
  from { opacity: 0; transform: translateY(30px) scale(.95); }
  to   { opacity: 1; transform: translateY(0)    scale(1);   }
}

/* 其他可选关键帧（按需使用） */
@keyframes fadeInUp    { from { opacity:0; transform:translateY(40px)  } to { opacity:1; transform:translateY(0) } }
@keyframes fadeInLeft  { from { opacity:0; transform:translateX(-50px) } to { opacity:1; transform:translateX(0) } }
@keyframes fadeInRight { from { opacity:0; transform:translateX(50px)  } to { opacity:1; transform:translateX(0) } }
@keyframes scaleIn     { from { opacity:0; transform:scale(.5)  } to { opacity:1; transform:scale(1) } }
@keyframes blurIn      { from { opacity:0; filter:blur(12px)    } to { opacity:1; filter:blur(0)    } }
@keyframes zoomInUp    { from { opacity:0; transform:scale(.6) translateY(40px) } to { opacity:1; transform:scale(1) translateY(0) } }
```

### 5. 常驻持续动画类（切换时需重置重启）

```css
/* 示例：脉冲、浮动等持续动画 —— 切换后由 JS 重置重启 */
@keyframes pulse    { 0%,100%{transform:scale(1)} 50%{transform:scale(1.06)} }
@keyframes float    { 0%,100%{transform:translateY(0)} 50%{transform:translateY(-12px)} }

.pulse      { animation: pulse  2.5s ease-in-out infinite; }
.float      { animation: float  3s   ease-in-out infinite; }
.float-slow { animation: float  5s   ease-in-out infinite; }
```

---

## 二、JavaScript 完整代码

```javascript
/* ========================================================
   snow-slides-switch — 页面切换引擎
   ======================================================== */

let current = 0;
let transitioning = false;

const slides = document.querySelectorAll('.slide');
const total  = slides.length;

// 顺序切换时轮换使用的效果列表
const TRANSITIONS = [
  'trans-left', 'trans-right', 'trans-up', 'trans-down',
  'trans-zoom', 'trans-fade', 'trans-blur', 'trans-scale-e'
];

// ⚠️ 需要在 applyAnimations 中重置的持续动画类（按实际使用的填写）
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
 * 切换到指定页面
 * @param {number} n    目标页面索引
 * @param {number} dir  方向：1=前进, -1=后退, 0=直接跳转
 */
function showSlide(n, dir) {
  if (transitioning) return;
  transitioning = true;

  const old = slides[current];
  const idx = (n + total) % total;
  const nu  = slides[idx];

  if (idx === current) { transitioning = false; return; }

  // 1. 隐藏当前页面（移除 active，触发旧页面的 CSS transition 淡出）
  old.classList.remove('active');

  // 2. 选择进场效果
  let transClass;
  if (dir === 1)       transClass = TRANSITIONS[idx % TRANSITIONS.length];
  else if (dir === -1) transClass = TRANSITIONS[(total - idx) % TRANSITIONS.length];
  else                 transClass = nu.getAttribute('data-trans') || 'trans-fade';
  transClass = 'trans-' + transClass.replace(/^trans-/, ''); // 统一加前缀

  // 3. 将新页面置于离屏位置并设为可见（不加 active）
  nu.classList.add(transClass);
  nu.style.display = 'flex'; // 使元素参与布局，才能触发 transition

  // 4. 双层 rAF：确保浏览器先渲染"离屏状态"，再启动 transition
  requestAnimationFrame(() => {
    requestAnimationFrame(() => {
      // 5. 加 active → CSS transition 自动将新页面从离屏动画到可见位置
      nu.classList.add('active');

      // ⚠️ 关键：在切换动画开始后 50ms 触发元素入场动画
      // 两者并行播放，避免切换结束后元素再次播放入场动画（二次动画）
      setTimeout(() => applyAnimations(nu), 50);

      // 6. 切换动画结束后清理（600ms CSS 时长 + 50ms 缓冲）
      setTimeout(() => {
        nu.classList.remove(transClass);
        old.style.display = 'none';
        current = idx;
        updateUI(); // 更新进度条、页码等 UI（自行实现）
        transitioning = false;
      }, 650);
    });
  });
}

/**
 * 重置并重播目标页面内所有动画
 * 核心：先 animation='none' + 强制回流，再重新赋值，使动画从头播放
 */
function applyAnimations(slide) {
  // — 重置分批入场元素（.anim-stagger + data-d 延迟属性）
  const staggerEls = slide.querySelectorAll('.anim-stagger');
  staggerEls.forEach((el, i) => {
    el.style.animation = 'none';
    el.offsetHeight; // 强制浏览器回流，清除旧动画状态
    const d     = parseFloat(el.getAttribute('data-d') || 0);
    const delay = d * 500 + i * 45; // ms；可按需调整节奏
    el.style.animation = `fadeInStagger .5s cubic-bezier(.25,.46,.45,.94) ${delay}ms both`;
    //                                                                      ↑ 'both' 保证动画前元素不闪烁
  });

  // — 重置所有常驻持续动画（进入页面时重新从头循环）
  slide.querySelectorAll(CONTINUOUS_ANIM_CLASSES).forEach(el => {
    el.style.animation = 'none';
    el.offsetHeight;
    el.style.animation = ''; // 清空内联样式，让 CSS class 动画重新生效
  });
}

// ——— 导航快捷方式 ———
function nextSlide() { showSlide(current + 1,  1); }
function prevSlide() { showSlide(current - 1, -1); }

// 键盘 & 滚轮导航
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

// 触屏滑动
let touchStartY = 0;
document.addEventListener('touchstart', e => { touchStartY = e.touches[0].clientY; });
document.addEventListener('touchend',   e => {
  const diff = touchStartY - e.changedTouches[0].clientY;
  if (Math.abs(diff) > 50) { diff > 0 ? nextSlide() : prevSlide(); }
});

// 初始化
window.addEventListener('load', () => {
  setTimeout(() => applyAnimations(slides[0]), 100);
});
```

---

## 三、HTML 使用规范

### 幻灯片标记模板

```html
<!-- data-trans: 直接跳转（dir=0）时使用的过渡效果 -->
<!-- trans-* 类：同时作为该页面的默认 CSS 初始状态标记 -->
<div class="slide active" data-slide="0" data-trans="blur">
  <!-- 元素入场动画：添加 .anim-stagger 类 + data-d 属性（延迟秒数）-->
  <h1 class="anim-stagger" data-d="0">标题</h1>
  <p  class="anim-stagger" data-d="0.15">副标题，延迟 150ms 出现</p>
  <div class="anim-stagger" data-d="0.3">内容块，延迟 300ms 出现</div>
</div>

<div class="slide" data-slide="1" data-trans="right">
  <h2 class="anim-stagger" data-d="0">第二页标题</h2>
</div>

<div class="slide" data-slide="2" data-trans="zoom">
  <!-- 持续动画类（.pulse .float 等）在页面切换时会自动重置重播 -->
  <div class="anim-stagger float" data-d="0">🚀</div>
</div>
```

### `data-trans` 可用值

| 值         | 效果           | 描述                            |
|------------|----------------|---------------------------------|
| `left`     | 从左侧滑入     | `translateX(-100%) → 0`         |
| `right`    | 从右侧滑入     | `translateX(100%) → 0`          |
| `up`       | 从顶部滑入     | `translateY(-100%) → 0`         |
| `down`     | 从底部滑入     | `translateY(100%) → 0`          |
| `zoom`     | 缩小淡入       | `scale(0.7) → scale(1)`         |
| `scale-e`  | 弹性缩放淡入   | `scale(0.3) → scale(1)`         |
| `flip`     | Y 轴 3D 翻转   | `rotateY(90deg) → rotateY(0)`   |
| `fade`     | 纯淡入         | `opacity 0 → 1`                 |
| `blur`     | 模糊淡入       | `blur(12px)+opacity 0 → 正常`   |

### `data-d` 延迟规范

```
data-d="0"    → 立即入场（切换后 50ms 内）
data-d="0.1"  → 延迟约 100ms
data-d="0.3"  → 延迟约 200ms（常用于正文）
data-d="0.5"  → 延迟约 300ms（常用于末尾装饰）
```

公式：`实际延迟(ms) = data-d × 500 + 元素序号 × 45`

---

## 四、防二次动画机制详解

### 问题根源

若在切换动画**结束后**再触发元素入场动画，用户会看到：
1. 页面滑入（600ms）
2. 页面静止约 0ms
3. 元素再次做入场动画

这就是"二次动画"——视觉上有明显停顿感，体验割裂。

### 解决方案：时间重叠

```
showSlide() 调用时序：

requestAnimationFrame × 2
  └── nu.classList.add('active')       ← 切换动画开始
      setTimeout(applyAnimations, 50)  ← 50ms 后开始元素动画（overlap！）
      setTimeout(cleanup, 650)         ← 650ms 后清理

[CSS transition: 600ms]  ████████████████████████████
[元素动画起始偏移: 50ms]      ░░░░░░░░░░░░░░░░░░░░░░░░░░░░
                         ↑50ms
```

关键点：
- 元素动画使用 `animation-fill-mode: both` → 动画开始前元素处于 "from" 状态（不可见），不会出现内容闪现
- 元素动画时长（500ms）+ 分批延迟最长约 350ms，合计在 650ms 切换窗口内完成
- `el.style.animation = 'none'` + `el.offsetHeight`（强制回流）确保每次切换时动画从头重播，不受上一次状态影响

### 常驻动画的正确重置方式

```javascript
// ✅ 正确：清空内联样式，让 CSS class 重新生效（从头循环）
el.style.animation = 'none';
el.offsetHeight; // 回流
el.style.animation = ''; // 不是 'none'，是空字符串！

// ❌ 错误：不做回流，浏览器可能不重置
el.style.animation = 'none';
el.style.animation = '';
```

---

## 五、快速集成清单

- [ ] 复制 **CSS 基础布局** + **切换效果类** + **关键帧** 到 `<style>`
- [ ] 复制 **JavaScript 切换引擎** 到 `<script>`（替换 `updateUI()` 为实际实现）
- [ ] 所有 `.slide` 元素添加 `data-trans` 属性
- [ ] 需要入场动画的元素添加 `.anim-stagger` 和 `data-d` 属性
- [ ] 第一页添加 `class="slide active"`，其余页只有 `class="slide"`
- [ ] 确认 `CONTINUOUS_ANIM_CLASSES` 列表包含项目中使用的所有持续动画类名
