# Edge rendering vs. static vs. serverful trade‑offs

If you’re building for the web, you’ve basically got three ways to get pixels to people: ship static HTML, run a tiny function at the edge, or run a fuller app in a region. Here’s a quick guide to when each one actually makes sense.

What these terms actually mean
- Static: HTML is produced ahead of time at build (SSG) or periodically (ISR), optionally with Partial Prerendering (PPR) to stream dynamic islands later.
- Edge rendering: code runs in a distributed edge runtime (V8 isolates or WASM) close to users. Great for low‑latency logic; constrained by cold‑start budgets, CPU/memory limits, and platform APIs.
- Serverful SSR: traditional regional servers/functions. Full Node/Deno/Java runtimes, higher resource ceilings, and easy private networking. Latency is bound by geography and queueing.

Quick decisions
- Do you need the same bytes for everyone for hours? Go static.
- Do you need per‑request personalization or geo routing in <50 ms compute? Go edge.
- Do you need heavy libs (Puppeteer, headless PDF, ML), transactions, or VPC? Go serverful.
- Can you split the page? Static outer shell + streamed dynamic islands is often the best of both worlds.

When static wins
- Marketing, docs, blogs, product pages that change on schedule.
- Search pages whose results are stable for minutes and can be revalidated on tag.
- Predictable, cheap, resilient: CDN caches hide origin incidents.

If you go static
- Prefer content APIs that can be pulled at build and tagged for ISR invalidation.
- Use cache hints and revalidation consistently:
  - Next.js: fetch(url, { next: { revalidate: 3600, tags: ["posts"] } })
  - Invalidate on mutation: revalidateTag("posts") or revalidatePath("/blog")
- With PPR, pre‑render the shell and stream dynamic holes on demand to keep TTFB low while data stays fresh.

When edge wins
- Geo‑aware pricing, inventory, and copy; country/state VAT; locale bootstrapping.
- Logged‑out personalization (e.g., cart by cookie, AB variant) that doesn’t require sensitive data access.
- Request shaping: quickly reject bots, rate‑limit, or rewrite without a regional hop.

Edge gotchas to keep in mind
- Keep compute sub‑10–20 ms; avoid N+1 to distant origins.
- Avoid large native deps and Node‑only APIs; prefer Web standard APIs, fetch, and lightweight logic.
- Design for statelessness. If you need session data, use signed tokens or a fast edge KV/Cache layer.

When serverful wins
- Authenticated dashboards with complex joins and transactional writes.
- Heavy tooling: image/video processing, headless browsers, AI inference beyond tiny models.
- Enterprise integrations inside private networks (databases over VPC, SAP/Oracle, internal APIs).

Serverful gotchas
- Expect 100–400 ms extra round‑trip latency for global users; reduce by regionalizing where practical.
- Invest in backpressure, pooling, and circuit breakers; failures are less forgiving than at the edge/CDN.

Performance, cost, and complexity at a glance
- Static
  - Perf: Best TTFB globally; zero server compute on hit.
  - Cost: Lowest; bandwidth‑heavy, compute‑light.
  - Complexity: Low; invalidation and build pipelines are the main moving parts.
- Edge
  - Perf: Excellent when logic is local and data is cached nearby; degrades if reaching regional databases.
  - Cost: Pay per‑request compute; can be efficient for small functions, expensive for long CPU.
  - Complexity: Medium; platform constraints + observability across many pops.
- Serverful
  - Perf: Consistent but bound by region; can be fast for users near the region.
  - Cost: Higher baseline (instances) or bursty (functions) plus egress to CDN.
  - Complexity: Highest; infra, security, scaling, and networking.

How to mix them in real life
- Static shell + dynamic holes (PPR/streaming): Render nav/footer/above‑the‑fold statically; stream user‑specific data as it becomes available.
- Edge gate, serverful backend: Do auth cookie verification and AB at the edge, forward to a regional app for heavy lifting.
- Cache‑through on edge: Fetch regional data once, cache at the edge with short TTLs and tag‑based busting.


Anti‑patterns
- Doing all SSR at the edge while your database lives in one region — you’ll pay the RTT tax on every request.
- Making everything static and relying on client hydration to “fix” data freshness — you’ll ship too much JS and still show stale UI.
- Pushing heavy Node‑only libraries to edge runtimes — you’ll hit platform limits and cold starts.

Rule of thumb
- Start static where you can, go edge when you must, fall back to serverful when you need power.

A simple decision flow
1) Can it be the same for everyone for N minutes? Static/ISR.
2) If not, can logic run fast without private data? Edge.
3) If not, does it need heavy compute or private networks? Serverful.
4) Compose: static shell + streamed/edge islands + serverful APIs.

