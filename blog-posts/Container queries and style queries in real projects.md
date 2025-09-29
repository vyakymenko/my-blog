---
title: "CSS 2025: Container queries and style queries in real projects"
date: 2025-09-29
author: Valentyn Yakymenko
origin: human
ai_generated: false
---

# CSS 2025: Container queries and style queries in real projects

If you’re like me, you’ve probably wrestled with components that look perfect in one column and fall apart in another. The first time I flipped a layout with just a container width—no media queries—I felt like I’d been cheating the system all these years. Container queries finally let our components listen to the space they actually get.

When container queries graduated from “someday” to “widely shipping,” responsive CSS changed for good. In 2025, you can design and ship components that adapt to their parent, not the viewport, and you can conditionally style components based on the styling of their container too. This post is a field guide to using size queries, style queries, and container units in real projects — including patterns, migration strategies, and pitfalls to avoid.

What you’ll learn:
- When to use container queries vs. traditional media queries
- How to set up containment correctly (and safely) in production
- Practical size queries for cards, navigation, data tables, and complex layouts
- Real‑world style queries for theme variants and density modes
- Container units (cqw/cqh/cqi/cqmin/cqmax) and when they’re better than vw/vh
- Progressive enhancement and fallbacks so nothing breaks in older browsers
- Performance, testing, and debugging tips


1) The problem container queries solve
Viewport media queries are coarse. If a card moves from a wide main column to a tight sidebar, its rules shouldn’t depend on the device width. Component size and style should drive behavior. Container queries make components portable and truly reusable.

- Size queries: Style a component based on the container’s inline/block size.
- Style queries: Style a component based on certain styles of its container (e.g., theme or density via custom properties, or other queryable discrete properties where supported).


2) Enabling container queries safely
To query a container, you must designate an ancestor as a “container.” Do this on the component’s immediate wrapper or on a meaningful ancestor that defines available space.

Basic setup:
```css
/* 1) Make a container by type (the safest default is inline-size) */
.card-grid {
  container-type: inline-size; /* enables @container width queries for descendants */
  container-name: cards;       /* optional but helpful for named queries */
}

/* 2) Now descendants can use @container */
@container cards (width > 42rem) { /* or (inline-size > 42rem) */
  .card { grid-template-columns: 2fr 3fr; }
}
```

Rules of thumb:
- Prefer container-type: inline-size on layout wrappers. It avoids unintended fragmentation and is compatible with most patterns. Reserve container-type: size for very specific cases where block-size also matters.
- Name containers you’ll query from multiple places: container-name: sidebar; Then use @container sidebar (...).
- Don’t over‑nest container contexts unnecessarily. Each container establishes a new query context; keep it predictable.


3) Real‑world size query patterns
A. Responsive card that reflows media and content
```html
<section class="card-grid">
  <article class="card">
    <img class="card__media" src="cover.jpg" alt="" />
    <div class="card__body">
      <h3>Title</h3>
      <p>Supporting copy…</p>
      <a class="button" href="#">Read more</a>
    </div>
  </article>
  <!-- repeat cards -->
</section>
```

```css
.card-grid {
  display: grid;
  gap: 1rem;
  grid-template-columns: repeat(auto-fill, minmax(18rem, 1fr));
  container-type: inline-size;
  container-name: cards;
}

.card {
  display: grid;
  grid-template: "media" auto "body" 1fr / 1fr;
  background: #fff;
  border-radius: .75rem;
  overflow: clip; /* hide media overflow on rounded corners */
}

/* At wider container widths, switch to a side-by-side layout */
@container cards (width >= 30rem) {
  .card { grid-template: "media body" auto / 2fr 3fr; }
  .card__media { block-size: 100%; object-fit: cover; }
}

/* Add a larger layout tier when the grid itself gets roomy */
@container cards (width >= 60rem) {
  .card { grid-template-columns: 1.5fr 2fr; }
}
```

B. Sidebar navigation that upgrades from icons to labeled items
```html
<aside class="app-sidebar">
  <nav class="nav">
    <a class="nav__item" href="#"><i class="i-home"></i><span>Home</span></a>
    <a class="nav__item" href="#"><i class="i-analytics"></i><span>Analytics</span></a>
    <a class="nav__item" href="#"><i class="i-settings"></i><span>Settings</span></a>
  </nav>
</aside>
```

```css
.app-sidebar {
  inline-size: 3.5rem; /* collapsed by default (just icons) */
  container-type: inline-size;
  container-name: sidebar;
}

.nav__item { display: grid; grid-auto-flow: column; gap: .75rem; align-items: center; }
.nav__item span { display: none; }

@container sidebar (width >= 10rem) {
  .nav__item span { display: inline; }
}
```

C. Data table that progressively reveals columns
```css
.table-wrap { container-type: inline-size; }
.table { display: grid; grid-template-columns: 1fr 2fr; }

@container (width >= 30rem) { /* no name → nearest container by default */
  .table { grid-template-columns: 1fr 2fr 1fr; }
}
@container (width >= 50rem) {
  .table { grid-template-columns: 2fr 2fr 1fr 1fr; }
}
```


4) Style queries: component variants without extra classes
Style queries let you write rules that respond to the styles of the container. In practice today, the most reliable approach is to query custom properties that you set on the container. This keeps the API explicit and works across composition boundaries.

Example: card density and theme driven by container style
```html
<section class="card-grid" style="--density: compact; --theme: surface">
  ...
</section>
```

```css
.card-grid { container-type: inline-size; }

/* Opt into a denser layout when the container declares it */
@container style(--density: compact) {
  .card { padding: .75rem; gap: .5rem; }
  .card h3 { font-size: 1rem; }
}

/* Theme variant controlled by container */
@container style(--theme: surface) {
  .card { background: #fff; color: #111; }
}
@container style(--theme: elevated) {
  .card { background: #fff; box-shadow: 0 2px 8px rgba(0,0,0,.1); }
}
```

Notes on style queries in 2025:
- Custom properties are the most portable way to express container state (theme, density, contrast mode, etc.).
- Some engines support style queries for a subset of discrete properties (e.g., overflow: auto). Verify current support before relying on non‑custom‑property queries; keep a class or attribute‑based fallback if needed.
- Keep queries readable and minimal. One or two style gates per component is usually enough.


5) Container units: cqw, cqh, cqi, cqmin, cqmax
Container query units scale measurements to the query container instead of the viewport.

- 1cqw = 1% of the container’s inline size
- 1cqh = 1% of the container’s block size
- 1cqi = 1% of the smaller of the container’s inline/block size (in logical flow)
- cqmin/cqmax = min/max of the two axes

Practical uses:
```css
.hero { container-type: inline-size; }
.hero__title { font-size: clamp(1.25rem, 3cqw + .75rem, 3rem); }
.card { padding: clamp(.75rem, 2cqw, 2rem); }
```

Choose container units when the component should scale inside different columns or sidebars regardless of viewport. Prefer viewport units for page‑level hero sections that truly depend on viewport.


6) Progressive enhancement and fallbacks
You can ship container queries today with guard rails.

- Feature queries: Wrap advanced rules in @supports to avoid parse errors on very old engines.
- Fallbacks first: Write a solid baseline without queries; then layer enhancements.
- Don’t hide content that depends on a query. Degradation should be cosmetic, not functional.

Example scaffold:
```css
/* Baseline (works everywhere) */
.card { display: grid; gap: 1rem; }

/* Enhance with container queries when supported */
@supports (container-type: inline-size) {
  .card-grid { container-type: inline-size; container-name: cards; }
  @container cards (width >= 30rem) { .card { grid-template-columns: 2fr 3fr; } }
}
```


7) Architecture: where to place containers and queries
- Layout wrappers define containers; components consume them.
  - Good: .sidebar, .content, .card-grid wrappers with container-type set.
  - Avoid putting container-type on deeply nested nodes that don’t control space.
- Co-locate rules: Keep a component’s @container blocks near the base component styles for maintainability.
- Name important containers you reference from multiple components.
- Limit tiers: Two or three size breakpoints per component is often enough. Too many tiers complicate testing.


8) Migration: from media queries to container queries
Common refactor pattern for a section component:

Before (viewport media query):
```css
.section__content { display: grid; grid-template-columns: 1fr; }
@media (min-width: 900px) {
  .section__content { grid-template-columns: 1fr 2fr; }
}
```
After (container query):
```css
.section { container-type: inline-size; }
.section__content { display: grid; grid-template-columns: 1fr; }
@container (width >= 40rem) {
  .section__content { grid-template-columns: 1fr 2fr; }
}
```

Refactor tips:
- Convert breakpoint px to rem tied to your root font-size so components scale with user preferences.
- Choose container breakpoints that reflect component needs (e.g., when labels wrap) rather than viewport sizes.
- Keep some page‑level media queries for global layout shifts (e.g., turning the whole app from single‑ to two‑column at xl).


9) Accessibility and UX considerations
- Respect user font-size: use rem in thresholds to preserve behavior when users zoom.
- Hit targets: at smaller container widths, increase vertical spacing or switch to stacked layouts to preserve touch targets.
- Color contrast: when using style queries for theme variants, ensure contrast stays compliant in all variants.
- Motion: if you animate layout transitions across container widths, honor prefers-reduced-motion.


10) Performance: cost and hygiene
Container queries are designed to be efficient, but any conditional layout logic can cost if abused.

- Keep containers purposeful. Avoid attaching container-type to hundreds of tiny nodes.
- Prefer inline-size over size when block size isn’t truly needed.
- Avoid thrashing: Don’t toggle container-type dynamically in JS. Toggle classes or custom properties on the container instead.
- Audit in DevTools: Chrome DevTools highlights container query evaluation; watch for excessive reflows.


11) Debugging and testing
- Visualize containers: temporary outline helps find the active container context.
```css
/* debug only */
*[container-type] { outline: 1px dashed color-mix(in srgb, rebeccapurple 40%, transparent); }
```
- Unit test thresholds: If you use component tests (Playwright/Vitest), render components in controlled width wrappers and assert class/role/visibility outcomes.
- Storybook: Add “container width” and style knobs to stories so designers can preview tiers.


12) Interop with modern CSS features
- Subgrid: Combine container queries with subgrid to keep columns aligned while letting components rearrange within their allocated tracks.
- :has(): Useful for stateful thresholds, e.g., cards that switch layout when they contain an image or a long label — combine with size queries for robust behavior.
- View transitions: When tiers switch, wrap structural changes with polite transitions; avoid disorienting large jumps.


13) Browser support (early 2025 snapshot)
- Size queries (@container width/inline-size, etc.): broadly supported in current Chrome, Edge, Safari, and Firefox stable.
- Style queries (@container style(...)):
  - Supported in Chromium‑based browsers and recent Safari for custom property queries; broader property support varies.
  - Firefox support is in progress/behind a flag in some versions. Verify before relying on non‑custom‑property queries.
- Container units (cqw, cqh, cqi, cqmin, cqmax): shipping in Chromium and Safari; Firefox support landing/behind a flag in some channels.

Always verify with caniuse.com or MDN for the exact versions you target, and gate advanced bits with @supports.


14) Patterns you’ll reuse
- Card media flip: Stack → side‑by‑side at 30rem container width.
- Menu labeling: Icons only → icons + labels at 10rem container width.
- Dense mode: @container style(--density: compact) to tighten spacing.
- Themed surfaces: @container style(--theme: elevated) for shadowed cards inside certain sections.
- Scalable type: clamp() with cqw for headings that fit their column.


15) A small, shippable starter
```html
<section class="feature-list">
  <article class="feature">
    <h3>Fast</h3>
    <p>Blazing performance at scale.</p>
  </article>
  <article class="feature">
    <h3>Secure</h3>
    <p>Privacy‑first by design.</p>
  </article>
  <article class="feature">
    <h3>Open</h3>
    <p>Built on web standards.</p>
  </article>
</section>
```

```css
.feature-list {
  display: grid;
  gap: 1rem;
  grid-template-columns: repeat(auto-fill, minmax(16rem, 1fr));
  container-type: inline-size;
  container-name: features;
}
.feature { padding: 1rem; border: 1px solid #ddd; border-radius: .5rem; }

@container features (width >= 28rem) {
  .feature-list { gap: 1.25rem; }
  .feature { padding: 1.25rem; }
}

/* Optional density via style query */
@container style(--density: compact) {
  .feature { padding: .75rem; }
}

/* Scalable heading with container units */
.feature h3 { font-size: clamp(1rem, 2.5cqw + .5rem, 1.5rem); }
```


16) FAQs
Q: Do I still need media queries?
A: Yes. Keep page‑level layout and OS‑level preferences (dark mode, reduced motion) in media queries. Use container queries inside components.

Q: Should I use container-type: size everywhere?
A: No. Start with inline-size. Use size only when the block axis must be considered, and you’ve validated the layout.

Q: Can I nest containers?
A: Yes, but be intentional. Each @container without a name matches the nearest container ancestor. Use names when crossing component boundaries.

Q: Are style queries stable enough for production?
A: For custom property queries, yes in most evergreen browsers. For querying other properties, verify support and keep fallbacks.


References and further reading
- MDN: Container queries overview — https://developer.mozilla.org/docs/Web/CSS/CSS_container_queries
- Spec drafts (CSS Containment Module Level 3/4)
- “Style Queries” explainer — https://github.com/w3c/csswg-drafts/issues/6399
- Can I use: container queries, style() queries, container units — https://caniuse.com
- Chromium and WebKit release notes for latest property support

If you adopt only one thing this quarter, make it this: move responsive logic into the component. Container and style queries make your UI more portable, robust, and easier to design. Ship them behind @supports, and you’ll be future‑proof without leaving any users behind.

— Thanks for reading. If you ship something cool with container or style queries, I’d love to hear about it. Ping me on Twitter/X or drop me an email to make this guide better.
