# Bun and Deno in FE workflows; Node innovations (test runner, loaders, ESM)

Front-end workflows are evolving fast. Three JavaScript runtimes meaningfully shape day-to-day DX: Node.js, Bun, and Deno. Node is modernizing with a built-in test runner, first-class ESM, and user-land loaders; Bun and Deno ship batteries-included developer ergonomics like fast dev servers, native TypeScript, and integrated test tooling.

This article is a practical guide: when to pick which, how to compose them in monorepos, and concrete code you can paste into real projects.

What you’ll learn:
- How Bun and Deno improve FE workflows (dev server, test runner, bundling, TypeScript, npm interop)
- What’s new in Node (node:test, ESM-by-default patterns, loaders/hooks)
- Side-by-side code examples for Node, Bun, and Deno
- Migration and coexistence tips in polyglot repos


1) Quick positioning: when each runtime shines
- Node.js
  - Strengths: ecosystem gravity (npm), long-term stability, mature production story, wide platform support.
  - Today’s upgrades: built-in node:test, stable ESM patterns, loaders for transformation/resolution.
  - Use when: serverful apps, CLI tools, broad library compatibility, enterprise infra.
- Bun
  - Strengths: very fast startup and test runs, built-in bundler/transformer, sensible web APIs, good npm compatibility, file watching.
  - Use when: fast FE dev server, tests on large React/Vue codebases, single-binary DX for local tooling.
- Deno
  - Strengths: native TypeScript, secure-by-default permissions, URL/JSR imports, web-standard APIs, excellent test tooling and built-in linter/formatter.
  - Use when: standards-first code, TS-heavy repos, edge-ready APIs, strong security controls.

Rule of thumb
- If your repo is npm-centric and must match existing Node production: use Node, optionally Bun for local dev/test speed.
- If you want TypeScript without build steps and a batteries-included toolchain: Deno is ergonomic.
- Polyglot is normal: you can run FE dev with Bun or Deno and still ship Node to prod.


2) Project setup patterns
A. Node (ESM project)
package.json:
```json
{
  "name": "fe-app",
  "type": "module",
  "scripts": {
    "dev": "node --watch ./dev-server.js",
    "test": "node --test",
    "build": "node ./scripts/build.mjs"
  },
  "devDependencies": {
    "vite": "^5.4.0"
  }
}
```

B. Bun
```bash
bun init
bun add -d vite
bun dev         # runs vite if defined in package.json scripts
bun test        # runs bun's test runner
```
Example package.json for Bun dev:
```json
{
  "scripts": {
    "dev": "vite",
    "test": "bun test",
    "build": "vite build"
  }
}
```

C. Deno
No package.json required. Use deno.json for tasks and permissions.
```json
{
  "tasks": {
    "dev": "deno run --watch --allow-net --allow-read dev_server.ts",
    "test": "deno test",
    "build": "deno run -A scripts/build.ts"
  },
  "lint": { "rules": { "tags": ["recommended"] } },
  "fmt": { "useTabs": false, "indentWidth": 2 }
}
```
Run tasks:
```bash
deno task dev
deno task test
```


3) Testing: node:test vs bun:test vs deno test
A. Node’s built-in test runner
- Node 20+ ships node:test. No external framework required.
- Supports subtests, TAP output, parallelization, watch mode via --test --watch.

Example (ESM):
```js
// math.test.mjs
import test from 'node:test';
import assert from 'node:assert/strict';

function add(a, b) { return a + b; }

test('add adds numbers', () => {
  assert.equal(add(2, 2), 4);
});

test('async example', async (t) => {
  await t.test('nested subtest', () => {
    assert.ok(true);
  });
});
```
Run:
```bash
node --test
node --test --watch  # great for local TDD
```

B. Bun’s test runner
- Jest-like API with fast start, snapshots, and built-in mocking.
```ts
// math.test.ts
import { describe, it, expect } from 'bun:test';

function add(a: number, b: number) { return a + b; }

describe('math', () => {
  it('adds', () => {
    expect(add(1, 2)).toBe(3);
  });
});
```
Run:
```bash
bun test
bun test --watch
```

C. Deno test
- Batteries-included; uses standard assertions and permissions model.
```ts
// math_test.ts
import { assertEquals } from "jsr:@std/assert";

function add(a: number, b: number) { return a + b; }

Deno.test("adds", () => {
  assertEquals(add(2, 3), 5);
});
```
Run:
```bash
deno test
deno test --watch
```

Snapshot testing
- Bun: built-in snapshots with expect(...).toMatchSnapshot().
- Deno: snapshot libraries exist on JSR; use permissions with care.
- Node: use third-party snapshot libs or tap via reporters.


4) TypeScript and module resolution
- Node
  - ESM preferred for modern tooling. Use "type": "module" or .mjs extensions.
  - TS needs a transpile step or a loader (tsx, swc). Examples below.
- Bun
  - TS/TSX supported out of the box; no ts-node needed. Good JSX support for React/Vue.
- Deno
  - Native TypeScript with zero config. Imports can be bare (via npm:) or URL/JSR.

Examples
A. Node with tsx for dev
```json
{
  "type": "module",
  "devDependencies": { "tsx": "^4.19.0" },
  "scripts": { "dev": "tsx watch src/index.ts" }
}
```

B. Node ESM + custom loader to import TS
```js
// ts-loader.mjs (minimal demo – use in dev only)
import { transformSync } from 'esbuild';

export async function load(url, context, defaultLoad) {
  if (url.endsWith('.ts')) {
    const { source } = await defaultLoad(url, { ...context, format: 'module' });
    const { code } = transformSync(source.toString(), { loader: 'ts', format: 'esm' });
    return { format: 'module', source: code };
  }
  return defaultLoad(url, context, defaultLoad);
}
```
Run with Node loaders:
```bash
node --loader ./ts-loader.mjs src/index.ts
```

C. Deno bare TS ESM
```ts
// src/mod.ts
export const greet = (name: string) => `Hello, ${name}`;
```
Run:
```bash
deno run src/mod.ts
```

D. Bun TS
```ts
// src/index.ts
export const greet = (name: string) => `Hello, ${name}`;
```
Run:
```bash
bun run src/index.ts
```


5) Loaders, hooks, and import maps
A. Node ESM loaders (resolution/transform hooks)
- Use --loader to intercept resolution and transform sources (e.g., TS, CSS modules for SSR, SVG-to-URL).
- Newer Node also supports --import to preload modules before user code.

Resolution example (alias @/* -> ./src/*):
```js
// alias-loader.mjs
export async function resolve(specifier, context, next) {
  if (specifier.startsWith('@/')) {
    const url = new URL(specifier.replace('@/', './src/'), import.meta.url);
    return { url: url.href, shortCircuit: false }; // defer to next for format
  }
  return next(specifier, context, next);
}
```
Run:
```bash
node --loader ./alias-loader.mjs app.mjs
```

Preload example:
```bash
node --import=dotenv/config --test   # preload dotenv for tests
```

B. Deno import maps
- Configure path aliases and external URLs in deno.json.
```json
{
  "imports": {
    "@/": "./src/",
    "lodash": "npm:lodash@^4"
  }
}
```
Usage:
```ts
import { debounce } from "lodash";
import { foo } from "@/utils/foo.ts";
```

C. Bun aliasing
- Bun respects tsconfig paths and supports import maps via bunfig.toml.
```toml
# bunfig.toml
[install]
peer = true

[alias]
"@/" = "./src/"
```


6) Dev servers and bundling for FE apps
- Node: pair with Vite, Webpack, or Rspack. Start via node or tsx and defer to the bundler.
- Bun: ships a fast bundler/transformer and can run Vite or its own Bun.serve() for simple apps.
- Deno: use deno serve for static + SSR, or integrate with Vite via plugins; Fresh (Deno’s framework) is zero-build for islands.

Minimal HTTP dev servers
A. Node (ESM + undici)
```js
// dev-server.mjs
import { createServer } from 'node:http';

createServer((req, res) => {
  res.setHeader('content-type', 'text/plain');
  res.end('Hello from Node dev server');
}).listen(5173, () => console.log('http://localhost:5173'));
```

B. Bun
```ts
// dev_server.ts
export default {
  port: 5173,
  fetch(_req: Request) {
    return new Response('Hello from Bun dev server', { headers: { 'content-type': 'text/plain' } });
  }
};
```
Run:
```bash
bun run dev_server.ts
```

C. Deno
```ts
// dev_server.ts
Deno.serve((_req) => new Response('Hello from Deno dev server', { headers: { 'content-type': 'text/plain' } }));
```
Run:
```bash
deno run --allow-net dev_server.ts
```


7) npm, JSR, and compatibility
- Node: native npm. ESM-first packages are common; prefer import semantics in new code.
- Bun: excellent npm compatibility; uses the same package.json and node_modules. Often faster installs.
- Deno: can import npm: specifiers without node_modules. For example:
```ts
import express from "npm:express@^4"; // yes, for quick adaptation; prefer standard fetch/Request in Deno
```
- JSR: the JavaScript Registry for standards-based modules. Deno and Node can both consume JSR packages with tooling; prefer JSR for TS+ESM libraries.

Publishing libraries
- If targeting all three runtimes, avoid Node-only globals. Stick to Web APIs (fetch, URL, TextEncoder) or provide conditional exports.
- For Node, define exports with ESM/CJS conditions in package.json.
```json
{
  "name": "lib",
  "type": "module",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.js",
      "require": "./dist/index.cjs"
    }
  }
}
```


8) Security and permissions
- Node: no permission sandbox; rely on process sandboxing and least-privilege at OS/container level.
- Bun: similar to Node (no explicit permission flags), but fewer legacy attack surfaces.
- Deno: permissioned by default. Grant only what you need per command:
```bash
deno run --allow-net=api.example.com --allow-read=./data app.ts
```


9) Mixing runtimes in monorepos
- Use workspaces (pnpm/npm/yarn) for Node/Bun packages; keep Deno projects side-by-side with their own deno.json.
- Provide universal scripts via make or justfile, or use task runners that shell out to each runtime.
- Example layout:
```
repo/
  apps/
    web-node/        # Next.js (Node)
    web-bun/         # Vite + Bun dev
    web-deno/        # Fresh (Deno)
  packages/
    ui/              # ESM library, Web APIs only
```
- Contract: packages/ui must avoid Node-only modules. Test in all runtimes:
```bash
node --test packages/ui/*.test.mjs
bun test packages/ui
deno test packages/ui
```


10) Node innovations in focus
A. node:test as a daily driver
- Stable, fast, built-in. Pair with --experimental-test-coverage in newer Node when available, or use c8.

B. ESM done right
- Prefer "type": "module". For CJS interop, use createRequire or dynamic import.
```js
// cjs-interop.mjs
import { createRequire } from 'node:module';
const require = createRequire(import.meta.url);
const legacy = require('left-pad');
```

C. Loaders and preloading
- --loader for custom resolution/transform; --import for preloading env and hooks.
- Keep production loaders minimal; prefer build-time transforms. Use loaders for SSR glue (CSS modules, SVG, TS in dev).


11) Choosing quickly (cheat sheet)
- Need pure speed for local dev/tests in an npm repo? Try Bun for dev/test; deploy with Node.
- Need out-of-the-box TS, strict permissions, and web-standard APIs? Use Deno end-to-end.
- Need maximum ecosystem compatibility and enterprise stability? Node remains the default.
- Building a UI library to run everywhere? Target Web APIs; run tests in all three.


12) Migration tips and pitfalls
- Avoid process, Buffer, and path-specific code in shared FE libraries; prefer URL, Blob, File, TextEncoder, crypto.subtle.
- For fetch differences, polyfill Request/Response types consistently (most are aligned now across runtimes).
- Node ESM gotchas: file URL to path conversion:
```js
import { fileURLToPath } from 'node:url';
import { dirname } from 'node:path';
const __filename = fileURLToPath(import.meta.url);
const __dirname = dirname(__filename);
```
- Bun and Deno ship fetch globally; Node 18+ does too, but older LTS needs undici.
- Snapshots: Bun built-in; Deno via libs; Node via external libs.


Closing thoughts
Bun and Deno push the envelope for developer experience—fast feedback loops, built-in TypeScript, and pragmatic defaults. Node keeps improving where it matters: standardized modules, first-party testing, and extensible loaders. In practice, you don’t have to pick a single winner. Use the best tool per task, keep shared code on Web APIs, and make testing across runtimes a habit.
