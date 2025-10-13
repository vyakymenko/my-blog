# Color contrast with OKLCH; prefers-reduced-motion and motion design ethics

Front-end for everyone and today we going discuss color contrasts with OKLCH, how to use it. You will understand prefers-reduced-motion and motion design ethics.

- Use OKLCH as your primary design-system color space. It tracks perceived lightness (L) and chroma (C) more linearly than HSL/HSV and sRGB, making contrast‑safe scaling and theming predictable.
- Build a contrast‑safe palette by locking L for text and adjusting C and h to meet WCAG 2.2 (4.5:1 body text, 3:1 large text/UI). Add an APCA/optional contrast target for future‑proofing.
- Progressive enhance: ship solid sRGB fallbacks first; layer OKLCH via @supports and color‑mix in oklch.
- Motion is a product ethic. Default to motion that communicates, not decorates. Respect prefers-reduced-motion at the component level, not just globally. Provide non‑motion alternatives for essential feedback.
- Create a “motion budget” and an a11y test plan. Keep heavy motion off critical paths. Verify with real users and tooling (Lighthouse, Web Vitals, reduced‑motion audits).

1) Why OKLCH now
HSL and HSV are convenient but not perceptually uniform: equal steps in “lightness” don’t look equal. sRGB is a device‑dependent encoding, not a design space. OKLCH (Oklab in polar form: Lightness L [0–1], Chroma C [0–~0.37], Hue h [0–360]) gives you visually even steps and predictable tints/shades across hues.

What this unlocks
- Contrast-consistent scales: Hold L for text roles; vary C and h for branding without breaking legibility.
- Safer theming: Adjust a small set of tokens by L and C to switch light/dark/high‑contrast modes without re‑authoring palettes.
- More color gamut: On wide‑gamut displays, OKLCH maps well while remaining backwards compatible.

Quick primer
- L (lightness): perceived brightness. Biggest lever for contrast.
- C (chroma): colorfulness/saturation. High C can reduce contrast at the same L against some backgrounds.
- h (hue): angle around the color wheel.

Diagram (conceptual)

   OKLCH (cylindrical)
        ↑ L
        │
        │       (higher chroma →)
        ●───◯───◯───◯  h rotates around
       /      
      /   C
     ⟳ h

2) Contrast fundamentals that matter in production
- WCAG 2.2 requires 4.5:1 for body text < 18pt (or < 24px) and 3:1 for large text/UI icons/graphics. Non‑text elements also need sufficient contrast when conveying meaning.
- APCA (WCAG 3.0 candidate) uses a perceptual model better aligned with human vision, especially for low‑contrast UIs on modern displays. It’s not a normative requirement yet. I recommend: satisfy WCAG 2.2; track APCA deltas in CI for future readiness.
- Don’t chase a single “ratio” blindly. Real context matters: weight, font rendering, size, background noise, and motion.

3) Practical OKLCH palette building for contrast
Approach
- Decide roles first: text/foreground (primary, secondary, disabled), backgrounds (surface, elevated), borders, states (info, success, warn, danger), and accent.
- Lock foreground L targets by role and vary C slightly to keep brand feel.
- Derive backgrounds by stepping L up/down; keep chroma low for surfaces to avoid color cast banding.

Example tokens (progressive enhancement)

CSS with safe sRGB fallback, upgraded to OKLCH when supported.

:root {
  /* sRGB fallbacks (closest approximations) */
  --fg-strong: #0b0b0c;        /* ≈ oklch(0.15 0.02 260) */
  --fg: #222326;               /* ≈ oklch(0.30 0.03 260) */
  --fg-muted: #5a5d65;         /* ≈ oklch(0.55 0.02 260) */
  --bg: #ffffff;               /* ≈ oklch(0.97 0 0) */
  --bg-soft: #f6f7f9;          /* ≈ oklch(0.96 0.01 260) */
  --accent: #005fcc;           /* ≈ oklch(0.60 0.14 255) */
  --accent-contrast: #ffffff;  /* ≈ oklch(0.98 0 0) */
}

@supports (color: oklch(0.5 0.1 0)) {
  :root {
    /* OKLCH first-class tokens */
    --fg-strong: oklch(0.15 0.02 260);
    --fg: oklch(0.30 0.03 260);
    --fg-muted: oklch(0.55 0.02 260);
    --bg: oklch(0.97 0 0);
    --bg-soft: oklch(0.96 0.01 260);
    --accent: oklch(0.60 0.14 255);
    --accent-contrast: oklch(0.98 0 0);
  }

  .button {
    color: var(--accent-contrast);
    background: var(--accent);
    /* derive hover by mixing accent toward fg for better perceived contrast */
    background: color-mix(in oklch, var(--accent) 88%, var(--fg-strong));
  }
}

Contrast‑safe text heuristic
- For light backgrounds (~L ≥ 0.94), keep body text L ≤ 0.30 and chroma low (C ≤ 0.05) to reduce edge fringing on LCD subpixels.
- For dark mode (bg L ≤ 0.15), push text L ≥ 0.85; avoid neon C that glows on OLED.

4) Computing/validating contrast in pipelines
Browser‑native contrast functions like color-contrast() are still experimental. Today, measure contrast in CI using a library. Two options:

Node (culori) example

```ts
// tooling/contrast.ts
// npm i culori
import { converter } from 'culori';
import { wcagContrast } from 'culori/contrast';

const toRgb = converter('rgb');

function contrast(a: string, b: string) {
  const A = toRgb(a);
  const B = toRgb(b);
  return wcagContrast(A, B); // returns ratio
}

const pairs = [
  ['oklch(0.30 0.03 260)', 'oklch(0.97 0 0)'],
  ['oklch(0.98 0 0)', 'oklch(0.60 0.14 255)'],
];

for (const [fg, bg] of pairs) {
  console.log(fg, 'on', bg, '→', contrast(fg, bg).toFixed(2), ':1');
}
```

CI guardrail (pseudo)
- Parse CSS for tokens.
- Compute contrast for role‑based pairs.
- Fail PR if body text < 4.5:1 or large text < 3:1.
- Log APCA as info for future migration.

5) Theming with OKLCH without regressions

Light/dark + high‑contrast modes

:root { /* light */
  --bg: oklch(0.98 0 0);
  --fg: oklch(0.30 0.03 260);
}

@media (prefers-color-scheme: dark) {
  :root {
    --bg: oklch(0.12 0 0);
    --fg: oklch(0.86 0.02 260);
  }
}

/* User‑requested high contrast variant (via class) */
:root.hc {
  --fg: oklch(0.08 0.02 260);
  --bg: oklch(0.99 0 0);
}

Tip: Separate perceptual knobs
- L toggles readability.
- C carries brand feel, raise it in accents, lower it in text.
- h differentiates categories (e.g., success vs danger) while preserving contrast via L.

6) prefers-reduced-motion: from checkbox to engineering standard
Motion can communicate hierarchy, causality, and feedback. It can also make people sick or distracted. As a principle: motion must always increase clarity and never gate core tasks.

The baseline
- Respect prefers-reduced-motion globally and locally.
- Provide a motion budget per surface (e.g., onboarding: 200ms small fades; dashboard: zero ambient parallax).
- Avoid autoplaying motion on entry where possible. Delay non‑essential motion until user intent.

Global opt‑down guardrail

/* Keep CSS transitions but swap to opacity/transform with reduced amplitude */
@media (prefers-reduced-motion: reduce) {
  * {
    animation: none !important;
    scroll-behavior: auto !important;
    transition-duration: 0.01ms !important; /* effectively none */
    transition-delay: 0s !important;
  }
}

Component-level ethical defaults

/* Base */
.toast {
  transform: translateY(8px);
  opacity: 0;
  transition: transform 180ms ease-out, opacity 120ms linear;
}
.toast[open] { transform: translateY(0); opacity: 1; }

/**** Respect reduced motion without removing feedback ****/
@media (prefers-reduced-motion: reduce) {
  .toast {
    /* Skip spatial motion, keep clarity via cross‑fade */
    transform: none;
    transition: opacity 80ms linear;
  }
}

React 18+ example (Framer Motion)

```tsx
import { useReducedMotion, motion } from 'framer-motion';

export function Button({ children }) {
  const shouldReduce = useReducedMotion();
  const variants = shouldReduce
    ? { initial: { opacity: 0 }, animate: { opacity: 1 } }
    : { initial: { y: 8, opacity: 0 }, animate: { y: 0, opacity: 1 } };

  return (
    <motion.button initial="initial" animate="animate" variants={variants}>
      {children}
    </motion.button>
  );
}
```

Angular example (Angular animations)

```ts
import { Component } from '@angular/core';
import { trigger, transition, style, animate } from '@angular/animations';

@Component({
  selector: 'app-toast',
  standalone: true,
  template: `<div [@appear] class="toast"><ng-content /></div>`,
  animations: [
    trigger('appear', [
      transition(':enter', [
        style({ opacity: 0, transform: 'translateY(8px)' }),
        animate('180ms ease-out', style({ opacity: 1, transform: 'none' }))
      ])
    ])
  ]
})
export class ToastComponent {}
```

Respect user preference in Angular globally by swapping to NoopAnimationsModule for a reduced-motion build or read the media query in a service and vary animation metadata accordingly.

JS utility to read the user preference

```ts
export const prefersReducedMotion = () =>
  typeof window !== 'undefined'
    ? window.matchMedia('(prefers-reduced-motion: reduce)').matches
    : false;
```

7) Motion ethics: what not to do (and why)
- Don’t use parallax/zoom on scroll for primary content. Vestibular issues are real; sudden z‑axis motion can trigger nausea.
- Don’t gate trust behind motion. Loading skeletons are fine; fake progress or distracting shimmer loops are not.
- Don’t couple motion and color extremes. High‑chroma flicker plus motion is a migraine recipe.
- Don’t surprise. Any motion exceeding ~200ms or > 12px travel should be intent‑driven with visible affordances.

Do instead
- Use subtle opacity/scale under 1.04, distances under 8–12px, duration 120–200ms for UI feedback.
- Prefer cross‑fade over translate/rotate for reduced motion contexts.
- Annotate animations with purpose (enter, exit, emphasis, spatial continuity) and measure their value.

8) Performance and Web Vitals
- Stick to compositor‑only properties (opacity, transform). Avoid layout‑thrashing properties in transitions (top/left, width/height) unless measured and isolated.
- Avoid heavy animated SVG filters and large GIF/WebM loops; they can tank CPU and battery.
- Audit CLS: sudden content shifts from lazy images/ads are a form of harmful motion.
- Test both normal and reduced‑motion modes in Lighthouse and WebPageTest. Treat regressions as bugs.

9) Real‑world rollout plan (what I do with teams)
- Inventory colors. Convert existing tokens to OKLCH (one‑time script with culori). Store both representations; OKLCH is authoritative.
- Define target L ranges by role. Example: body text L≈0.25–0.32 (light mode), L≈0.85–0.92 (dark mode). Accents keep brand hue but adjust C per background.
- Implement fallbacks and @supports upgrade. Ship color-mix(in oklch, ...) only in supporting browsers.
- Add CI contrast checks (WCAG now, APCA info). Break builds on regressions.
- Create a motion policy. Include: prefers‑reduced‑motion handling, motion budget, banned patterns (parallax), and required alt patterns.
- Refactor animation utilities/components to consult reduced‑motion at runtime.
- Run an accessibility review with users. Watch for motion discomfort reports; adjust defaults.

10) Common pitfalls and fixes
- “My brand blue looks dull in dark mode.” Fix: Raise C slightly but increase L contrast; don’t push neon C on dark backgrounds.
- “Text on accent buttons fails contrast.” Fix: Nudge button L downward and enforce white text; alternatively desaturate accent slightly at the same L.
- “Animations feel sluggish on low‑end devices.” Fix: Shorten duration, prefer opacity, drop blur/backdrop filters, reduce simultaneous animated elements.
- “We ‘support’ reduced motion but skeletons still shimmer.” Fix: Provide a static placeholder path when reduced motion is on.

Appendix A: Relative color syntax + color-mix examples

/* Derive a subtle border from foreground in OKLCH */
@supports (color: oklch(0.5 0.1 0)) {
  .card {
    --border: color-mix(in oklch, var(--fg) 15%, var(--bg));
    border: 1px solid var(--border);
  }
}

/* Tone an accent for hover/focus */
.button:hover {
  background: color-mix(in oklch, var(--accent) 82%, var(--fg-strong));
}

Appendix B: ASCII palette visualizer (L vs C)

L: 0.98 bg  ────────────────────────────────●
L: 0.86 text-dark  ────────────────────●
L: 0.60 accent  ─────────────● (C↑)
L: 0.30 text  ─────● (C low)
L: 0.12 bg-dark  ●

Final checklist
- OKLCH tokens with sRGB fallbacks
- @supports upgrade path + color-mix in oklch
- WCAG 2.2 contrast CI guards; APCA logged informationally
- Reduced-motion respected globally and per component with non‑motion alternatives
- Motion budget documented; banned patterns enforced
- Performance profiled for both normal and reduced-motion modes

Further reading/tools
- OKLCH guide: https://oklch.com/
- Culori: https://culorijs.org/
- APCA: https://www.myndex.com/APCA/
- Web Vitals: https://web.dev/vitals/
- WCAG 2.2: https://www.w3.org/TR/WCAG22/

