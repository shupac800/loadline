---
name: teachme
description: Generate a complete interactive educational webapp with multiple lessons, interactive controls, canvas visualizations, and a sandbox mode — for any subject. Invoke with /teachme [subject].
argument-hint: "[subject description]"
user-invocable: true
allowed-tools: Read Write Bash Edit Glob Grep Agent WebSearch WebFetch
effort: max
---

# TeachMe — Interactive Lesson Webapp Generator

Generate a **single self-contained HTML file** that teaches `$ARGUMENTS` through an interactive, multi-lesson webapp with real-time visualizations and hands-on controls.

## Reference implementation

Read `index.html` in the project root — that is the reference app (a triode load-line tutorial). Study its architecture thoroughly before generating anything. Every pattern described below comes from that file.

## What to generate

A single `index.html` file (or a name the user specifies) containing **all HTML, CSS, and JavaScript inline** — zero external dependencies. The app must:

1. **Teach a subject** through 6–15 progressive lessons, each building on the previous
2. **Visualize** the subject's core model on an interactive canvas (NW quadrant)
3. **Expose controls** (sliders, radio buttons, toggles) that manipulate the model in real time (NE quadrant)
4. **Show lesson content** with rich HTML — formulas, inline SVG diagrams, styled lists (SW quadrant)
5. **Display secondary visualizations** — graphs, metrics, waveforms — that react to the same model (SE quadrant)
6. **Gate controls per lesson** — each lesson locks irrelevant controls and unlocks the ones the student should explore
7. **Offer a Sandbox mode** that unlocks everything for free exploration

---

## Architecture (match the reference exactly)

### Layout: 2×2 grid

```
┌─────────────────────────┬──────────────────────────┐
│  Primary visualization  │  Controls & settings     │
│  (quad-nw, canvas)      │  (quad-ne, scrollable)   │
├─────────────────────────┼──────────────────────────┤
│  Lesson text + nav      │  Secondary visualizations│
│  (quad-sw, scrollable)  │  (quad-se, scrollable)   │
└─────────────────────────┴──────────────────────────┘
```

### CSS requirements

- Dark theme using CSS custom properties (`:root` variables)
- Color palette: `--bg: #15181d`, `--panel: #1d2128`, `--panel-2: #262b33`, `--line: #2f3540`, `--text: #d8dde5`, `--text-dim: #8a93a3`, `--accent: #ffd34d` plus 2-4 subject-specific accent colors (e.g. for traces, highlights, warnings)
- `body`: flex column, 100vh, overflow hidden
- `header`: flex row with title, subtitle, Lesson/Sandbox tabs
- `main`: CSS grid `1fr 1fr / 1fr 1fr`, 12px gap, fills remaining height
- Each `.quad`: panel background, 1px border, 8px radius, flex column
- `.quad.scroll`: overflow-y auto
- Responsive: `@media (max-width: 1100px)` collapses to single column
- `.ctrl.locked`: opacity 0.4, pointer-events none
- Controls: `.ctrl > .row` (label + value display) + `<input type="range">` or radio group
- Buttons: dark panel background, accent for `.primary`
- Canvas elements: `width: 100%`, flex-grow in NW quad; fixed height in SE quad

### HTML structure

```html
<header>
  <h1>[Short Title]</h1>
  <span class="sub">[Subtitle]</span>
  <div class="tabs">
    <div class="tab active" id="tab-lesson">Lesson</div>
    <div class="tab" id="tab-sandbox">Sandbox</div>
  </div>
</header>

<main>
  <!-- NW: primary visualization canvas -->
  <div class="quad quad-nw">
    <canvas id="plot"></canvas>
    <div class="readout" id="readout"></div>  <!-- overlay with key metrics -->
  </div>

  <!-- NE: controls -->
  <div class="quad quad-ne scroll">
    <h2>Controls</h2>
    <!-- Each control wrapped in <div class="ctrl" data-ctrl="keyName"> -->
    <!-- Contains: .row with label + .val span, then <input type="range"> -->
    <!-- Optional: .hint tooltip, radio groups, etc. -->
    <div class="signal-row">
      <button id="playBtn">Play</button>
      <button id="resetBtn">Reset to defaults</button>
    </div>
  </div>

  <!-- SW: lesson content -->
  <div class="quad quad-sw scroll lesson" id="lessonCard">
    <div class="lesson-nav">
      <button id="prevBtn">&larr; Prev</button>
      <span class="step" id="lessonStep"></span>
      <button id="nextBtn" class="primary">Next &rarr;</button>
    </div>
    <h3 id="lessonTitle"></h3>
    <div id="lessonBody"></div>
  </div>

  <!-- SE: secondary visualizations -->
  <div class="quad quad-se scroll">
    <!-- 1-3 additional canvases and/or metric panels -->
  </div>
</main>
```

### JavaScript architecture

#### State object
```js
const state = {
  mode: "lesson",     // "lesson" | "sandbox"
  lesson: 0,          // current lesson index
  playing: false,     // animation running?
  t0: 0, tNow: 0,    // animation timing
  // ... all model parameters as flat numeric/string fields
};
```

#### LESSONS array
```js
const LESSONS = [
  {
    title: "1. [Lesson title]",
    body: `<p>Rich HTML content...</p>`,  // supports <code>, <b>, inline SVG, formulas
    // Per-lesson visualization flags:
    show___: true/false,   // what to show/hide on the primary canvas
    // Control gating — true = locked (grayed out, disabled):
    lock: { controlKey1: true, controlKey2: false, ... },
    // Optional defaults applied when entering this lesson:
    defaults: { paramKey: value, ... },
  },
  // ...
];
```

#### Core patterns to implement

1. **`$(id)`** — shorthand for `getElementById`
2. **`bindSlider(id, stateKey, scale)`** — generic slider→state→redraw binding
3. **Logarithmic sliders** where needed (use `val = 10^(a + b*slider)` with inverse)
4. **`gotoLesson(i)`** — apply defaults, zero locked params, call applyLessonGates, redraw
5. **`applyLessonGates()`** — iterate all `[data-ctrl]` elements, toggle `.locked` class based on `LESSONS[state.lesson].lock`
6. **`setMode(m)`** — switch lesson/sandbox, show/hide lesson card, re-gate
7. **`syncLabels()`** — update all `.val` text displays from state
8. **`redraw()`** — call all draw functions
9. **`fitCanvas(c)`** — handle devicePixelRatio for crisp rendering on retina
10. **`togglePlay()` / `tick(now)`** — requestAnimationFrame animation loop
11. **`resetDefaults()`** — restore initial state, update all sliders
12. **`drawPlot()`** — primary visualization on NW canvas
13. **`drawReadout()`** — overlay metrics on NW canvas
14. **Secondary draw functions** for SE canvases

#### Canvas drawing conventions
- Use `fitCanvas()` before every draw (handles DPI scaling)
- Coordinate transforms: define `X(val)` and `Y(val)` mapping functions
- Grid lines: `strokeStyle = "#2c323d"`, labels in `#8a93a3`
- Axis frame: `strokeStyle = "#3a4250"`
- Data traces: use the accent colors, lineWidth 1.4–2
- For "glow" effects: stack 3-4 passes from wide+dim+blurry to narrow+bright+sharp
- Animated dots: `fillStyle = accent`, arc radius 4-5px
- Text labels: `11px ui-sans-serif, system-ui, sans-serif`

#### Init
```js
function init() {
  setMode("lesson");
  gotoLesson(0);  // or 1 if lesson 0 is just context-setting
  syncLabels();
  redraw();
  window.addEventListener("resize", redraw);
}
init();
```

---

## Subject-specific design process

When generating for a subject, think through:

1. **What is the core model?** (e.g., for circuits: Ohm's law + tube curves; for optics: Snell's law + ray tracing; for economics: supply/demand curves)
2. **What are the 4-8 key parameters?** These become sliders/controls
3. **What is the primary visualization?** The NW canvas should show the model's main diagram — the thing the student needs to build intuition about
4. **What are 1-3 secondary visualizations?** These go in the SE quad — time-domain plots, frequency analysis, phase diagrams, metric summaries, etc.
5. **What is the lesson progression?** Start with the simplest view (one concept, most controls locked), progressively unlock controls and introduce complexity
6. **What are the key "aha" moments?** Each lesson should have a clear insight the student discovers by manipulating the unlocked controls

### Lesson writing guidelines

- **Lesson 1**: Introduce the fundamental object/concept with no controls. Just look at it.
- **Lessons 2-3**: Introduce the first interactive element. Explain cause and effect.
- **Lessons 4-6**: Build up the core model. Each lesson adds one concept.
- **Lessons 7-9**: Combine concepts. Show emergent behavior.
- **Lessons 10+**: Advanced topics, edge cases, real-world applications.
- **Final lesson**: Full picture — all controls unlocked, all visualizations active.

Each lesson body should:
- Be 150-400 words of well-structured HTML
- Use `<code>` for variable names and formulas
- Use `<b>` for key terms on first introduction
- Include inline SVG diagrams where they aid understanding (simple, schematic-style)
- End with a clear instruction: "Move the **X** slider to see Y"
- Use `<sub>` and `<sup>` for subscripts/superscripts in formulas
- Use `&mdash;`, `&rarr;`, `&middot;`, etc. for proper typography

### Domain model guidelines

- Implement the actual math/physics/economics/etc. model, not a toy approximation
- Use bisection or Newton's method for implicit equations
- If the model has well-known named formulas or laws, implement them faithfully
- Include at least one nonlinear or emergent behavior that surprises the student
- The model should be fast enough for 60fps animation (pre-compute what you can)

---

## Quality checklist

Before declaring the file complete, verify:

- [ ] Opens in a browser with no console errors
- [ ] All 4 quadrants render correctly
- [ ] Lesson navigation (Prev/Next) works, step counter updates
- [ ] Controls lock/unlock correctly per lesson
- [ ] Sandbox mode unlocks all controls, hides lesson card
- [ ] Canvas visualizations are crisp (DPI-aware) and responsive to resize
- [ ] Sliders update visualizations in real time (no lag)
- [ ] Play/Pause animation works smoothly
- [ ] Reset button restores defaults
- [ ] Readout overlay shows key metrics, updates live
- [ ] At least 6 lessons with progressive complexity
- [ ] No external dependencies — everything is inline
- [ ] Dark theme looks polished — no white flashes, consistent spacing
- [ ] Responsive layout collapses gracefully below 1100px

---

## Output

Generate the complete HTML file. Do not create placeholder content — every lesson must have real, accurate, well-written educational content. Every visualization must implement the actual model math. The result should be a genuinely useful teaching tool, not a demo skeleton.

Write the file, then open it in the browser to verify it works. Fix any issues before reporting completion.
