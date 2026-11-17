# Next.js 16 migration: the five failures that actually stop a build

## 1. Identify which migration broke you

Next.js 16 finished the migrations that 15 only warned about. Read the first error line and jump
straight to that section — the errors look like unrelated bugs because most of them surface at
request time, not build time.

| Error / symptom | Cause in 16 | Section |
|---|---|---|
| `Route "/x/[slug]" used "params.slug" directly. "params" is a Promise` | Synchronous access to `params` / `searchParams` removed | [2](#2-async-request-apis) |
| `Cannot read properties of undefined (reading 'slug')` at request time | Same, but the value is read three files away from the route | [2](#2-async-request-apis) |
| `Webpack is configured while Turbopack is not` | Turbopack is the default for `next build`; a leftover `webpack` key fails the build on purpose | [3](#3-turbopack-vs-a-webpack-config) |
| `The "middleware" file convention is deprecated` | `middleware.ts` renamed to `proxy.ts`, Node runtime only | [4](#4-middleware--proxyt) |
| Build error naming a `@slot` route | Every parallel route slot needs its own `default.js` | [5](#5-parallel-route-slots) |
| `dynamic`, `revalidate`, `fetchCache` now error | `cacheComponents: true` rejects the old segment config | [6](#6-cache-components) |
| `next lint` not found | The command was removed from the CLI | [7](#7-small-breaks) |

Minimum versions moved too: **Node.js 20.9+** and **TypeScript 5.1+**. A CI image still on Node 18
fails before any of the above appears.

## 2. Async request APIs

`params`, `searchParams`, `cookies()`, `headers()` and `draftMode()` are Promises. In 15 a thenable
proxy resolved them synchronously; 16 removed the proxy, so `params.slug` is `undefined` and the
error appears wherever the value is finally used.

```tsx
// server component — await, and let the generated type name the route
export default async function Page(props: PageProps<'/blog/[slug]'>) {
  const { slug } = await props.params;
  const { q } = await props.searchParams;
  return <Article slug={slug} q={q ?? ''} />;
}
```

```tsx
// client component — use() during render only; it throws inside an event handler
'use client';
import { use } from 'react';

export default function ClientPage({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = use(params);
  return <h1>{slug}</h1>;
}
```

```ts
const token = (await cookies()).get('session')?.value;
const ua = (await headers()).get('user-agent');
```

- `npx next typegen` writes `.next/types` (`PageProps`, `LayoutProps`, `RouteContext`). Without it,
  `PageProps<'/route'>` does not exist and people fall back to hand-written `params: Promise<...>`
  types that drift from the route.
- Prefer awaiting in the server component and passing the resolved value down. A `use()` call in a
  client tree adds a suspending boundary during hydration for no benefit.
- The codemod (`npx @next/codemod@canary next-async-request-api .`) comments out what it cannot
  prove safe. Search the diff for `UnsafeUnwrapped` before trusting it.

If the build succeeds and the page 500s on the first request with `undefined`, the cause is almost
always one metadata or layout file still reading `props.params` synchronously.

## 3. Turbopack vs a webpack config

```js
// next.config.js — fails "next build" in 16
module.exports = {
  webpack: (config) => {
    config.module.rules.push({ test: /\.svg$/, use: '@svgr/webpack' });
    return config;
  },
};
```

Two ways out:

```js
// A. stay on webpack for now
// package.json: "build": "next build --webpack"

// B. move the loader to Turbopack rules
module.exports = {
  turbopack: {
    rules: {
      '*.svg': { loaders: ['@svgr/webpack'], as: '*.js' },
    },
  },
};
```

If you never wrote a `webpack` key, a plugin did (`next-compose-plugins`, older Sentry/Analyze
wrappers, MDX setups). `next build --webpack` is the safe rollback while you find it; it is not a
long-term home, because the build-speed work in 16 lands in Turbopack only.

## 4. Middleware → `proxy.ts`

```ts
// proxy.ts — Node.js runtime, same handler shape
import { NextResponse, type NextRequest } from 'next/server';

export function proxy(request: NextRequest) {
  if (!request.cookies.get('session') && request.nextUrl.pathname.startsWith('/app')) {
    return NextResponse.redirect(new URL('/login', request.url));
  }
  return NextResponse.next();
}

export const config = { matcher: ['/app/:path*'] };
```

| Trap | What happens | Fix |
|---|---|---|
| File renamed, handler still `export function middleware` | A proxy file with no proxy: every request passes through, no warning | Rename the export to `proxy` |
| `export const runtime = 'edge'` | Not supported in `proxy.ts` | Keep deprecated `middleware.ts`, or move the logic to Node |
| `skipMiddlewareUrlNormalize` | Config flag renamed to `skipProxyUrlNormalize`; the old key is ignored silently | Update `next.config` |
| Auth redirect "works locally, not in prod" | The redirect ran in middleware that no longer executes | Check the network tab for the redirect status |

This is the most dangerous item on the list because nothing fails: the app builds, pages render,
and an auth gate simply stops existing.

## 5. Parallel route slots

```tsx
// app/@modal/default.tsx — required for every @slot in 16
export default function Default() {
  return null;
}
```

`return null` keeps the previous "render nothing" behaviour. `notFound()` inside the default makes a
hard navigation to the route 404 instead of rendering the slot empty — pick by what the slot means.

## 6. Cache Components

```tsx
// errors once cacheComponents: true
export const dynamic = 'force-dynamic';
export const revalidate = 60;
export const fetchCache = 'force-no-store';
```

```tsx
// function-level caching instead
import { cacheLife, cacheTag } from 'next/cache';

async function listPosts() {
  'use cache';
  cacheLife('minutes');
  cacheTag('posts');
  return db.post.findMany({ orderBy: { publishedAt: 'desc' } });
}
```

`cacheComponents: true` is a caching redesign, not an upgrade checkbox: data fetching becomes
dynamic by default and `use cache` marks what to keep. Turning it on to "finish" the migration
usually trades a build error for an unexpectedly slow site. `revalidateTag('posts')` also lost
stale-while-revalidate unless you pass a profile — `revalidateTag('posts', 'max')` — and `updateTag`
is the new primitive for read-your-own-writes.

## 7. Small breaks

| Removed / changed | Replacement |
|---|---|
| `next lint` | ESLint directly (`eslint .`), or Biome |
| AMP support | Drop it |
| `images.qualities` defaults | Declare every quality you use |
| `experimental.turbo` | Top-level `turbopack` key |
| Node 18, TypeScript 4.5 | Node 20.9+, TypeScript 5.1+ |

## 8. Order of operations that works

```bash
npx @next/codemod@canary upgrade latest      # package bump + official codemods
npx @next/codemod@canary next-async-request-api .
npx next typegen                            # generates PageProps / LayoutProps
npx tsc --noEmit                            # type errors left by the codemod
npm run build                               # Turbopack build (or --webpack to isolate)
```

1. Fix `params` / `searchParams` errors first — they cascade into confusing runtime failures.
2. Then rename `middleware.ts` → `proxy.ts` and change the export name.
3. Then add the missing `default.js` files.
4. Leave `cacheComponents` off until the build is green; it is a separate change.
5. Diff a production build's route list (`next build` output) against 15. Missing routes mean a
   slot default or a proxy matcher is wrong.

The longer walkthrough with worked examples is
[Next.js 16 upgrade: async params, proxy.ts, and Turbopack build breaks](https://www.ryuslog.com/blog/nextjs-16-upgrade-breaking-changes-async-params-proxy-turbopack).

While debugging the fallout, these run entirely in the browser:
[JSON formatter](https://www.ryuslog.com/tools/json) ·
[YAML ↔ JSON](https://www.ryuslog.com/tools/yaml) ·
[diff checker](https://www.ryuslog.com/tools/diff) ·
[timestamp converter](https://www.ryuslog.com/tools/timestamp) for cache and `Date` bugs ·
[cURL → fetch/Python/Go](https://www.ryuslog.com/tools/curl) when a request works in `curl` and not in the app.
