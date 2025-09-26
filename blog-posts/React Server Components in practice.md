# React Server Components in practice (Next.js App Router patterns, streaming, caching, partial pre‑render)

Updated: 2025‑09‑26

TL;DR
- React Server Components (RSC) let you render on the server by default in the Next.js App Router, shipping almost zero JS for those components and enabling data‑fetch close to the source.
- Use server components for data fetching, composition, and heavy dependencies; use client components only when you need interactivity, browser APIs, or local state with hooks.
- Stream with Suspense boundaries, loading.js, and nested layout/page segments to render critical UI first and progressively hydrate islands.
- Know your caches: request memoization, the data cache, the full route cache (static/PPR), and the router cache. Control them with fetch cache modes, revalidate options, and cache tags.
- Partial Prerendering (PPR) lets you statically pre‑render a shell while streaming dynamic holes on request. It gives the best of SSG’s TTFB and SSR’s freshness.


1) What are React Server Components (RSC)?
RSC allow components to run on the server at request time (or build/static time) and return a serialized description of UI to the client. In the App Router, all components are server components by default. Client components are opt‑in via the "use client" directive at the top of a file.

Key properties
- No client bundle for server components: they don’t ship JS to the browser.
- Async/await in components: you can await promises directly in server components.
- Component boundaries matter: a server component can render a client component as a child, but not the other way around.
- Data proximity: fetch from the server component; avoid API ping‑pong.

When to use which
- Server component (default): data fetching, heavy libs (DB/SDK), environment secrets, markdown/MDX processing, large lists.
- Client component ("use client"): interactivity (onClick, forms with local state), browser APIs (window, localStorage), animations, focus management.


2) App Router mental model and file system patterns
- app/layout.tsx: server component defining global chrome (header/footer). Can wrap children with Suspense for streaming.
- app/page.tsx: the root route’s server component. Nested routes use app/(group)/segment/page.tsx.
- app/loading.tsx: special component shown immediately while the nearest parent Suspense boundary streams children.
- app/template.tsx vs layout.tsx: template re‑renders per navigation; layout persists across navigations.
- Client islands: place "use client" at the top of files that need interactivity. Keep them leaf‑like.
- Route handlers: app/api/foo/route.ts for server‑only endpoints when needed.
- Server Actions: mutate data directly from a server component or client form without a bespoke API route.

Example structure
/app
  layout.tsx
  loading.tsx
  page.tsx
  (marketing)/page.tsx
  (app)
    dashboard
      layout.tsx
      loading.tsx
      page.tsx
      @charts/Chart.tsx       // use client
      @server/Revenue.tsx     // server component


3) Data fetching patterns in server components
Simple fetch with caching

```tsx
// app/dashboard/page.tsx (server component)
export default async function DashboardPage() {
  const res = await fetch("https://api.example.com/summary", {
    // default is 'force-cache' in RSC, but make it explicit for clarity
    cache: "force-cache",
    next: { revalidate: 3600, tags: ["summary"] },
  });
  const data = await res.json();
  return (
    <main>
      <h1>Dashboard</h1>
      <Summary data={data} />
    </main>
  );
}
```

Fresh on every request

```tsx
await fetch(url, { cache: "no-store" });
```

Time‑based revalidation (ISR)

```tsx
await fetch(url, { next: { revalidate: 300 } }); // 5 minutes
```

Tag‑based cache invalidation

```tsx
import { revalidateTag } from "next/cache";

// In a server action after a mutation
revalidateTag("summary");
```

Path revalidation

```tsx
import { revalidatePath } from "next/cache";
await revalidatePath("/dashboard", "page");
```

For per‑request memoization (avoid duplicate fetches within one render):
- fetch is request‑memoized automatically in the same context.
- You can also use the cache(fn) helper from react for idempotent computations.

```ts
import { cache } from "react";

export const getUser = cache(async (id: string) => {
  const res = await fetch(`https://api.example.com/users/${id}`, { cache: "force-cache" });
  return res.json();
});
```

Dynamic vs static segments
- Static by default if all data is cacheable (force-cache or revalidate) and no dynamic APIs (cookies(), headers()) are used.
- Mark a route dynamic at the segment level: export const dynamic = "force-dynamic"; or revalidate = 0.
- Or mark the whole route static: export const dynamic = "error"; or export const revalidate = 600.

```ts
// app/products/[id]/page.tsx
export const dynamic = "force-dynamic"; // always run on request
// OR
export const revalidate = 600;           // statically generate and revalidate
```


4) Streaming with Suspense, loading.js, and nested layouts
Streaming lets the server flush HTML for resolved parts of the tree while slower branches continue in the background.

Patterns
- Wrap slow children with Suspense to stream a placeholder immediately.
- Use app/segment/loading.tsx to show route‑level pending UI.
- Use nested layouts for progressive reveal across sections.

Example

```tsx
// app/layout.tsx (server)
import { Suspense } from "react";

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <Header />
        <Suspense fallback={<NavSkeleton />}>
          {/* Nav may read user and permissions */}
          <Nav />
        </Suspense>
        <main>{children}</main>
      </body>
    </html>
  );
}
```

```tsx
// app/dashboard/page.tsx (server)
import { Suspense } from "react";

export default function DashboardPage() {
  return (
    <>
      <h1>Dashboard</h1>
      <div className="grid">
        <Suspense fallback={<CardSkeleton title="Revenue" />}>
          {/* Streams when ready */}
          <Revenue />
        </Suspense>
        <Suspense fallback={<CardSkeleton title="Top Products" />}>
          <TopProducts />
        </Suspense>
      </div>
    </>
  );
}
```

```tsx
// app/dashboard/loading.tsx
export default function Loading() {
  return <PageSkeleton />; // immediate first paint while server streams
}
```

Notes
- Avoid putting "use client" above a large subtree; it blocks server streaming for that subtree.
- Keep client components as leaves, and pass data from server parents as props.


5) Partial Prerendering (PPR)
PPR renders a static shell at build time (or ISR) and treats some child regions as dynamic "holes" that stream on request. This gives near‑SSG TTFB for the shell while keeping dynamic sections fresh.

Enablement and behavior
- PPR support depends on your Next.js version. In recent versions with the App Router, PPR can be enabled per route or project. Check the Next.js docs for the version you use.
- A page becomes partially prerenderable when: the layout/template and immediate shell are static, and dynamic children are isolated behind Suspense boundaries.

Authoring for PPR
- Make the shell static by default (cacheable fetches, no cookies()/headers() in the shell).
- Wrap dynamic parts with Suspense so they can be deferred.
- Use revalidate or cache tags on dynamic parts to tune freshness.

Example

```tsx
// app/products/[id]/page.tsx
export const revalidate = 3600; // static shell with ISR

export default function ProductPage({ params }: { params: { id: string } }) {
  return (
    <>
      {/* Shell is static: title, meta, stable copy */}
      <ProductHero id={params.id} />
      {/* Dynamic holes stream on request: price, stock, personalized offers */}
      <Suspense fallback={<PriceSkeleton />}>
        <LivePrice id={params.id} />
      </Suspense>
      <Suspense fallback={<StockSkeleton />}>
        <StockLevel id={params.id} />
      </Suspense>
    </>
  );
}
```


6) Caching layers you should actually care about
- Request memoization: dedupes identical fetch() calls within the same request/render.
- Data Cache: persists fetch responses according to cache: and next: options. Revalidated by time or tags.
- Full Route Cache: the HTML/Flight payload for static/PPR output, served by the edge/node without recompute until revalidation.
- Router Cache (client): preserves component state between navigations and speeds up back/forward.

Choosing cache mode
- Global freshness required: cache: "no-store".
- Mostly static with acceptable staleness: next: { revalidate: N }.
- On‑demand invalidation after writes: next: { tags: [...] } + revalidateTag().
- Avoid cache stampede: prefer reasonable revalidate windows over no-store when possible; use suspense to stream.

Pitfall: mixing cookies()/headers() in a supposedly static shell marks the route dynamic and disables the full route cache. Isolate such reads to dynamic children behind Suspense.


7) Server Actions in practice
Server Actions let you mutate data without creating an API route and then revalidate.

```tsx
// app/actions.ts
"use server";
import { revalidatePath, revalidateTag } from "next/cache";

export async function saveSettings(input: FormData) {
  // perform mutation (DB call)
  // ...
  revalidatePath("/dashboard");
  revalidateTag("summary");
}
```

```tsx
// app/dashboard/SettingsForm.tsx
"use client";
import { experimental_useFormStatus as useFormStatus } from "react-dom";
import { saveSettings } from "../actions";

export function SettingsForm() {
  const { pending } = useFormStatus();
  return (
    <form action={saveSettings}>
      <input name="email" type="email" required />
      <button disabled={pending}>{pending ? "Saving..." : "Save"}</button>
    </form>
  );
}
```

Notes
- Mark server action files with "use server" at the top.
- Prefer tag/path revalidation that matches the data your page consumes.


8) Real‑world recipes
A. Personalized dashboard with fast TTFB
- Make layout and header static.
- Use loading.tsx for instant shell.
- Read user session only inside dynamic children (e.g., <Nav />), wrapped in Suspense.
- Stream slow widgets individually with Suspense; set per‑widget revalidate windows.

B. Product page SEO with live price
- Statically prerender product details from CMS (revalidate 24h).
- Stream <LivePrice> and <StockLevel> from commerce API with revalidate: 60.
- Invalidate price tag on updates from webhook via revalidateTag("product:123:price").

C. Search results
- Mark route dynamic with export const dynamic = "force-dynamic".
- Use cache: "no-store" for the search request.
- Stream filters sidebar (cacheable) and results list (dynamic) separately with Suspense.


9) Common pitfalls and how to avoid them
- Accidentally promoting a whole subtree to client: putting "use client" high in the tree disables RSC benefits for that subtree. Keep client components leaf‑level.
- Reading cookies/headers in layouts: this forces dynamic rendering and disables static/PPR. Read them only where truly needed.
- Coupling fetch freshness to UI: prefer decoupling via Suspense + streaming rather than making everything dynamic.
- Overusing revalidatePath: prefer fine‑grained tags so you don’t bust unrelated caches.
- Blocking user interaction while streaming: use optimistic UI in client components and let server stream confirmatory data.


10) Debugging and observability tips
- next dev shows which routes are static or dynamic during build/logs.
- Use React DevTools Profiler and the Next.js overlay to see Suspense waterfalls.
- Log cache headers on your API to validate caching behavior. Consider CDN caching for static assets and full route cache.


11) Migration notes (from Pages Router)
- Move data fetching from getServerSideProps/getStaticProps into server components.
- Replace client‑heavy pages with server components plus small client islands for interactivity.
- Adopt loading.tsx and Suspense to preserve UX during SSR/streaming.


12) Checklist
- Is the shell static or cacheable? If not, can I isolate dynamic bits behind Suspense?
- Are client components leaf‑like and minimal?
- Do I have clear revalidate windows or tags for each fetched dataset?
- Are Server Actions revalidating only what’s necessary?
- Are Suspense boundaries placed to maximize first paint and minimize waterfalls?

Further reading
- Next.js App Router docs
- React documentation on Server Components and Suspense
- Caching and Revalidation in Next.js

