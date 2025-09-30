---
title: "Angular 21 is coming: what I’m watching, how I’m preparing (and how you can too)"
date: 2025-09-29
author: Valentyn Yakymenko
origin: human
ai_generated: false
---

# Angular 21 is coming: what I’m watching, how I’m preparing (and how you can too)

Angular has been on a steady, predictable release cadence for years, with a focus on developer‑experience upgrades, performance (especially SSR + hydration), and modern primitives like signals and the new control flow. With v20 out earlier in 2025, the next stop is Angular 21.

I wrote this guide to help teams prepare pragmatically. It covers:
- What I expect to evolve in v21 based on recent trends and ongoing work
- Concrete code patterns that are already solid today and will continue to age well
- A migration checklist you can use to de‑risk your upgrade
- How to track breaking changes and features directly in the Angular repo

Note: I won’t make claims that require unretrievable context at publish time. Instead, I’ll point to where to verify details and show durable patterns that align with Angular’s direction.

Useful links:
- Angular on GitHub (PRs, discussions, roadmap): https://github.com/angular/angular
- Angular RFCs: https://github.com/angular/angular/discussions (filter by category “RFC”)
- Angular DevRel posts: https://blog.angular.io/


## A quick recap of where Angular is now

- Standalone is the default. New projects and guides revolve around standalone APIs, route‑level providers, and feature modules via directory structure—not NgModules.
- Signals and new control flow (@if, @for, @switch) are mainstream. Templates are cleaner, more predictable, and tree‑shakable.
- SSR + hydration are first‑class. Streaming and island‑style hydration patterns are encouraged; build tooling is optimized accordingly.
- Vite‑based builder and faster dev server workflows are the norm. DX continues to be a top priority.
- Router is fully functional, with lazy, defer, and fine‑grained preloading patterns. Typed APIs and route data are widely used.

If your app embraces the bullet points above, you’ve already done most of the work to make future upgrades smooth.


## What I’m watching for Angular 21

These are areas that have seen active iteration across the last few releases and are likely to keep improving. Treat them as a watch list; verify specifics in the Angular repo as v21 stabilizes.

1) Signals and zoneless change detection
- Continued polish around signal ergonomics in templates and host bindings
- Broader guidance for zoneless apps by default (fewer microtask surprises, more explicit change propagation)
- Tooling and schematics that make the opt‑in story simpler for large apps

2) SSR, hydration, and streaming
- More resilient hydration across complex component trees
- Clearer, faster defaults for streaming in common setups
- Better DX for debugging hydration mismatches

3) Router ergonomics and performance
- Incremental data loading, smarter preloading strategies, and improved error/loading UI patterns
- Typed redirects and cleaner route configuration ergonomics

4) Tooling and builder pipeline
- Devserver/builder iterations focused on faster feedback loops
- More consistent output chunking and better profiling hooks

5) CDK and Material (M3)
- Accessibility and theming refinements
- Better defaults for density, motion, and color systems aligned with Material 3

6) Typed forms, i18n, and testing
- Continued improvements in typed forms ergonomics
- I18n extraction/build stability for SSR
- Test harnesses and faster component tests with the new test APIs

Again: check the repo’s issue tracker and milestone board for the exact scope of v21 as it firms up.


## Durable patterns you can adopt now

These are safe, forward‑compatible patterns you can put in production today to be ready for v21.

1) Signals in components and templates

```ts
import { Component, signal, computed } from '@angular/core';

@Component({
  selector: 'app-cart',
  standalone: true,
  template: `
    <h2>Cart ({{ count() }})</h2>
    <ul>
      @for (item of items(); track item.id) {
        <li>
          {{ item.title }} — {{ item.price | currency }}
          <button (click)="remove(item.id)">Remove</button>
        </li>
      }
    </ul>
    <p>Total: {{ total() | currency }}</p>
  `,
})
export class CartComponent {
  readonly items = signal<{ id: number; title: string; price: number }[]>([]);
  readonly count = computed(() => this.items().length);
  readonly total = computed(() => this.items().reduce((s, i) => s + i.price, 0));

  add(item: { id: number; title: string; price: number }) {
    this.items.update(list => [...list, item]);
  }
  remove(id: number) {
    this.items.update(list => list.filter(i => i.id !== id));
  }
}
```

2) New control flow: @if, @for, @switch

```html
<!-- Cleaner and more powerful than *ngIf/*ngFor for complex UIs -->
@if (user()) {
  <h3>Welcome, {{ user()!.name }}</h3>
} @else {
  <p>Please sign in</p>
}

@switch (status()) {
  @case ('loading') { <spinner/> }
  @case ('ready')   { <dashboard/> }
  @default          { <p>Error</p> }
}
```

3) Defer non‑critical features

```html
<!-- Defer the heavy chart until it’s near viewport or idle -->
@defer (on viewport; prefetch on idle) {
  <app-heavy-chart />
} @placeholder {
  <skeleton-chart />
} @loading {
  <p>Loading chart…</p>
} @error {
  <p>Couldn’t load chart.</p>
}
```

4) SSR + hydration defaults

```bash
# Create a new app with SSR enabled
ng new my-app --ssr

# Or add SSR to an existing project
ng add @angular/ssr
```

Tips:
- Prefer streaming responses for data‑heavy pages; add clear loading/error UI using template control flow.
- Avoid hydration mismatches by keeping server and client rendering deterministic (no randoms in render, no Date.now() inside templates, etc.).

5) Functional router configuration with lazy boundaries

```ts
import { Routes } from '@angular/router';

export const routes: Routes = [
  {
    path: '',
    loadComponent: () => import('./home/home.component').then(m => m.HomeComponent),
  },
  {
    path: 'admin',
    loadChildren: () => import('./admin/routes').then(m => m.ADMIN_ROUTES),
    data: { preload: false },
  },
];
```

6) Route‑level providers and environment composition

```ts
import { provideHttpClient, withInterceptors } from '@angular/common/http';

export const routes: Routes = [
  {
    path: 'feature',
    providers: [
      provideHttpClient(
        withInterceptors([
          (req, next) => next(req.clone({ setHeaders: { 'X-Feature': '1' } }))
        ])
      )
    ],
    loadComponent: () => import('./feature/feature.component').then(m => m.FeatureComponent)
  }
];
```


## Migration checklist for v21 readiness

Use this as your pre‑upgrade punch list. The more boxes you check now, the easier v21 will be.

- Baseline
  - [ ] Project builds cleanly on the latest v20 with no deprecated APIs
  - [ ] Standalone APIs are used throughout; no new NgModules introduced
  - [ ] RxJS version aligns with Angular’s supported range; type errors resolved

- Templates and state
  - [ ] New control flow (@if/@for/@switch) used where appropriate
  - [ ] Signals adopted for local component state and derived values
  - [ ] @defer used for non‑critical or heavy islands

- Router and data
  - [ ] Routes are lazy by default with clear preloading policy
  - [ ] Route‑level providers encapsulate feature concerns (HTTP, interceptors)
  - [ ] Error and loading UIs are explicit and tested

- SSR and hydration
  - [ ] SSR is enabled for pages that benefit from SEO/TTFB
  - [ ] Streaming is used for heavy pages; hydration warnings are resolved
  - [ ] No non‑deterministic rendering on the server

- Tooling and tests
  - [ ] Vite/builder config is up‑to‑date and reproducible
  - [ ] Component tests use the latest test utilities/harnesses
  - [ ] E2E covers critical routes, including SSR paths

- Documentation
  - [ ] Internal ADR or upgrade notes exist for your repo
  - [ ] Third‑party libraries audited for Angular v20+ compatibility


## How I track real changes (so you can verify too)

When milestones and release candidates appear, I do the following in the Angular repo:

1) Watch the milestone board
- Go to https://github.com/angular/angular → Issues → Milestones
- Open the milestone for v21 (when created) to see scoping and progress

2) Filter PRs by label and milestone
- PRs → filter by milestone:"v21.x" and labels like feat, fix, perf, refactor, docs
- Scan for deprecations and breaking change labels

3) Read changelogs and release notes
- Check the Releases tab for RCs and final notes
- Verify breaking changes and required migrations

4) Validate locally before upgrading prod
- Spin up a reproduction branch
- Upgrade with ng update and run your test suite
- Fix deprecations incrementally and record notes for your team


## Frequently asked questions

What if I haven’t adopted standalone yet?
- Do it now. It’s the biggest long‑term simplifier. Convert feature‑by‑feature; don’t try to flip everything at once.

Do I have to adopt signals everywhere?
- No. Start with local component state and computed values. Interop with RxJS is fine; keep observable boundaries where they already model streams.

Will NgModules stop working?
- Legacy apps can keep running, but new guidance and APIs center on standalone. The farther you get from NgModules today, the easier your future.

How risky is the v21 upgrade?
- If you stay current (at v20) and resolve deprecations, upgrades are typically measured in hours to a few days for medium apps. The repo’s release notes will call out any breaking changes.


## Final thoughts

Angular 21 will continue the platform’s multi‑year trend: better ergonomics, smaller and faster apps, and clearer mental models for state and rendering. You don’t need to predict every detail to get ready. Adopt the durable patterns above, keep your app green on the latest major, and follow the repo to validate specifics as v21 ships.

If you spot an RFC or PR that materially changes this post’s guidance, ping me—I’ll update this article.
