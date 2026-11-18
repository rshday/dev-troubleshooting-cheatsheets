# Next.js in Docker: standalone output, Supabase cookies, and `redirect()` that never fires

## 1. Symptom → cause

| Symptom | Most likely cause | Section |
|---|---|---|
| Container serves HTML with no stylesheet, `/_next/static/…` is 404, dev is fine | `output: "standalone"` image copied only `.next/standalone` | [2](#2-what-standalone-actually-emits) |
| CSS loads, but `favicon.ico`, `og-image.png`, `robots.txt` are 404 | `public/` was never copied | [2](#2-what-standalone-actually-emits) |
| `Could not load the 'sharp' module using the linux-x64 runtime` | Native `@img/*` binary built for another libc or CPU | [3](#3-sharp-is-a-packaging-problem) |
| `Module 'sharp' not found` (Next error code `E47`) | `sharp` pruned by `--omit=optional`, or left in `devDependencies` | [3](#3-sharp-is-a-packaging-problem) |
| `sh: 1: next: not found` | `next start` in a standalone image | [4](#4-run-node-serverjs-not-next-start) |
| Health check fails from outside, `curl` inside the container succeeds | The server bound to something other than `0.0.0.0` | [4](#4-run-node-serverjs-not-next-start) |
| Signed in, dashboard renders, next refresh `getUser()` is `null` | Cookie adapter not `getAll`/`setAll`, or `cookies()` not awaited | [5](#5-the-supabase-server-client-is-a-cookie-adapter) |
| `Error: Cookies can only be modified in a Server Action or Route Handler` | `cookieStore.set()` during a render, swallowed by a `try/catch` | [5](#5-the-supabase-server-client-is-a-cookie-adapter) |
| `NavigatorLockAcquireTimeoutError: Acquiring an exclusive Navigator LockManager lock "lock:sb-<ref>-auth-token"` | Second Supabase client in the same request, or `supabase-js` < 2.107.0 | [6](#6-a-session-that-flips-between-signed-in-and-out) |
| Action validates, `redirect()` runs, browser stays on the form | `redirect()` inside `try/catch` — it navigates by **throwing** | [7](#7-server-actions-three-failures) |
| `413` / `Body exceeded 1mb limit` on upload | Action body limit (1 MB default), or the platform's own limit | [7](#7-server-actions-three-failures) |
| `useActionState` never updates, only a digest appears | Action returned or threw something not serialisable | [7](#7-server-actions-three-failures) |

## 2. What `standalone` actually emits

`output: "standalone"` changes two things: the build writes a runnable server, and tracing narrows
`node_modules` to the files actually imported. The emitted tree mirrors the **repository root**, not
your project directory — the single most common source of "the file exists but returns 404".

```
.next/standalone/
├── server.js            # entry point: node server.js
├── package.json         # minimal, no scripts
├── node_modules/        # traced runtime deps only
└── .next/
    ├── server/          # server chunks, route manifest
    └── required-server-files.json
.next/static/            # NOT included — copy it yourself
public/                  # NOT included — copy it yourself
.next/cache/             # NOT included — mount it if you use ISR
```

```dockerfile
# syntax=docker/dockerfile:1.7
FROM node:22-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci

FROM node:22-alpine AS build
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

FROM node:22-alpine AS run
WORKDIR /app
ENV NODE_ENV=production PORT=3000 HOSTNAME=0.0.0.0
COPY --from=build /app/.next/standalone ./
COPY --from=build /app/.next/static ./.next/static   # → /app/.next/static
COPY --from=build /app/public ./public
EXPOSE 3000
USER node
CMD ["node", "server.js"]
```

| Artefact | In `.next/standalone`? | Where it must land | Symptom when missing |
|---|---|---|---|
| `server.js` + traced `node_modules` | yes | `/app` | Container exits instantly |
| `.next/static/**` (CSS, JS, fonts) | no | `/app/.next/static` | Unstyled HTML, 404 on `/_next/static/*` |
| `public/**` | no | `/app/public` | Missing favicon, og-image, `robots.txt` |
| `.next/cache` (ISR output) | no | mount a volume or tmpfs | Every request re-renders |
| `next` CLI | no | not needed | `next: not found` |

- The copy target is relative to the **app root**, where `server.js` sits. `/app/static` or
  `/app/.next/standalone/.next/static` reproduces the same 404s you started with.
- Monorepo: with a workspace at `apps/web`, tracing writes
  `apps/web/.next/standalone/apps/web/server.js` plus a hoisted root `node_modules`. Pin the trace
  root (`outputFileTracingRoot`) or copy the whole `standalone` tree and keep `WORKDIR` at the root.
- Reverse proxy: Next prefixes assets with `/_next/`. A `try_files` that only routes `/` to the app,
  or one that strips a path prefix, breaks the exact URLs the HTML references. Proxy the whole host,
  or set `assetPrefix` so the emitted HTML points somewhere that answers.

## 3. `sharp` is a packaging problem

`sharp` is an optional dependency with platform binaries. Next 16 reports a missing module as
`Module 'sharp' not found` with error code `E47`; a **present but wrong-ABI** binary surfaces as
sharp's own `Could not load the 'sharp' module using the … runtime`.

```bash
docker run --rm my-app node -e "console.log(require('sharp').versions)"   # throws? wrong ABI
docker run --rm my-app ls node_modules/@img                              # empty? pruned
```

| Strategy | Survives host → container? | Image size | Notes |
|---|---|---|---|
| `npm ci` inside the runtime stage | yes | larger | Resolves the right `@img/sharp-*` package |
| Copy host `node_modules` | no (darwin/arm64 → linux) | small build cache | Fails on the first image request |
| `--os=linux --cpu=x64 --libc=musl` | partly | small | Must match the base image libc exactly |
| `images: { unoptimized: true }` | yes | smallest | No resizing/AVIF; browsers fetch originals |
| External loader (Cloudinary, imgix) | yes | small | Removes `sharp` from the critical path |

- `npm ci --omit=optional`, `.npmrc` with `optional=false`, or a Yarn `supportedArchitectures` entry
  all remove `@img/*` silently.
- Building on Apple Silicon for `linux/amd64` needs `--platform` on **every** stage; the correct
  optional package is chosen at install time and is never downloaded later.
- Alpine → musl; if libc drift keeps breaking the binding, `node:22-bookworm-slim` plus
  `libvips42` is the stable route.
- Keep `sharp` in `dependencies`, never `devDependencies`, so tracing and `npm ci --omit=dev` agree.

## 4. Run `node server.js`, not `next start`

The standalone tree contains no `next` CLI. The generated `server.js` reads `PORT`, `HOSTNAME` and
`NEXT_PRIVATE_*`, so:

```bash
# wrong in a standalone image
CMD ["npm", "run", "start"]        # → sh: 1: next: not found
# right
CMD ["node", "server.js"]
```

- `HOSTNAME` falls back to `0.0.0.0` in the generated server, but Docker exports `HOSTNAME` as the
  container id, so the container's own value wins. Set `ENV HOSTNAME=0.0.0.0` explicitly and verify
  from **outside** the container, not with `curl` inside it.
- `NODE_ENV` must be `production`; do not rely on the base image.
- ISR under `output: "standalone"` writes to `.next/cache` inside the container. With a read-only
  root or several replicas each pod renders separately and cold starts hit the database:

```yaml
services:
  web:
    image: my-app
    read_only: true
    tmpfs: ["/tmp", "/app/.next/cache"]
```

## 5. The Supabase server client is a cookie adapter

`@supabase/ssr` splits one logical client into a server shape with a cookie adapter and a browser
shape reading `document.cookie`. If the adapter is wrong the two halves disagree and the session
appears to flip between signed-in and signed-out. Since Next.js 15 `cookies()` is async, so a
synchronous adapter reads an empty store and the failure appears three files away.

```ts
// lib/supabase/server.ts — Next.js 15/16, @supabase/ssr 0.5+
import { createServerClient } from "@supabase/ssr";
import { cookies } from "next/headers";

export async function createClient() {
  const cookieStore = await cookies();          // Promise since Next.js 15

  return createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll: () => cookieStore.getAll(),      // every chunk, not just the base name
        setAll(cookiesToSet) {
          try {
            cookiesToSet.forEach(({ name, value, options }) =>
              cookieStore.set(name, value, options));
          } catch {
            // render: writes are illegal here — the proxy persists the refresh
          }
        },
      },
    },
  );
}
```

The `try`/`catch` here is correct **only** because something else performs the write. If nothing does,
the refresh is thrown away on every request — that is the `null` user on the second load.

In Next.js 16 the refresh belongs in `proxy.ts` (Node.js runtime only); `middleware.ts` still loads
but is deprecated and `runtime` can no longer be set. The proxy must write the cookies twice —
once onto the request for downstream reads, once onto the response for the browser:

```ts
// proxy.ts
export async function proxy(request: NextRequest) {
  let response = NextResponse.next({ request });
  const supabase = createServerClient(url, anonKey, {
    cookies: {
      getAll: () => request.cookies.getAll(),
      setAll(cookiesToSet) {
        cookiesToSet.forEach(({ name, value }) => request.cookies.set(name, value));
        response = NextResponse.next({ request });            // rebuild AFTER request writes
        cookiesToSet.forEach(({ name, value, options }) =>
          response.cookies.set(name, value, options));
      },
    },
  });

  const { data: { user } } = await supabase.auth.getUser();   // forces the refresh
  if (!user && request.nextUrl.pathname.startsWith("/dashboard")) {
    const url = request.nextUrl.clone();
    url.pathname = "/login";
    return NextResponse.redirect(url);
  }
  return response;                                            // must be the reassigned one
}
```

| Cookie adapter | Works in a render | Persists the refresh | Typical failure |
|---|---|---|---|
| `getAll` / `setAll` | yes, writes skipped | yes, via the proxy | none when the proxy returns its response |
| `get` / `set` / `remove` | partly | unreliable | Drops `…-auth-token.1` chunks |
| Omitted | reads `document` | never | `null` user on every request |
| Two clients per request | yes | no | `NavigatorLockAcquireTimeoutError` |

Cookies over roughly 4 KB are chunked into `sb-<ref>-auth-token.0`, `.1`. An adapter returning only
the base name returns half a session — that half-session is a JWT that verifies and is missing its
refresh token.

## 6. A session that flips between signed in and out

One log line separates the three breakages:

```ts
const { data: { session } } = await supabase.auth.getSession();
console.log({
  hasCookie: (await cookies()).getAll().some(c => c.name.startsWith("sb-")),
  sessionExpiry: session?.expires_at,
  now: Math.floor(Date.now() / 1000),
});
```

| Reading | Meaning |
|---|---|
| `hasCookie: false` | The adapter never saw the chunks (no `getAll`, or `cookies()` not awaited) |
| `hasCookie: true`, expiry in the past | The refresh happened but was never persisted |
| Both healthy, session `null` | A client created before the proxy ran |

- `getSession()` decodes the JWT without verifying it — fine for display, wrong for authorisation.
  `getUser()` calls the auth server. Use `getUser()` in the proxy and in every guard, and authorise
  against `user.id`.
- `@supabase/auth-js` serialises refresh through the Web Locks API; the auto-refresh tick asks for
  the lock with a 0 ms timeout, so it fails immediately when another holder is active. `2.107.0`
  removed the deadlock in the navigator-locks mutex; on older versions keep `onAuthStateChange`
  synchronous and defer other Supabase calls out of the callback.
- A Server Action runs after the proxy, so its cookies are fresh — but the tree it returns was
  rendered with the pre-action state. `revalidatePath("/", "layout")` before `redirect("/login")`,
  or sign-out works and the header keeps the avatar until a hard reload.

## 7. Server Actions: three failures

**`redirect()` throws.** It raises an error whose digest is
`NEXT_REDIRECT;<type>;<url>;<status>;` (`type` is `push` inside an action, `replace` during render;
status `307`, or `303`/`308` where the framework chooses). A `try/catch` around it swallows the
navigation and the action completes normally.

```ts
"use server";
export async function createPost(prev: State, formData: FormData): Promise<State> {
  try {
    if (!formData.get("title")) throw new Error("Title is required");
    await db.post.create({ data: { title: String(formData.get("title")) } });
  } catch (error) {
    return { ok: false, message: (error as Error).message };   // never rethrow
  }
  redirect("/posts");        // OUTSIDE the try — this line is the navigation
}
```

When the redirect genuinely belongs inside the guarded block, let the framework unwind:

```ts
import { unstable_rethrow } from "next/navigation";
try { await save(formData); redirect("/done"); }
catch (error) { unstable_rethrow(error); return { ok: false, message: "Save failed" }; }
```

The same trap removes navigation from `notFound()`, `forbidden()` and `unauthorized()`.

**The body limit is 1 MB.** The action parser rejects the whole multipart payload with HTTP 413 and
`Body exceeded 1mb limit.` Raise it in the top-level config key (Next 15+; it was `experimental` in
14), and remember a proxy can cap lower — Vercel will not accept a request body above 4.5 MB no
matter what `bodySizeLimit` says, so larger uploads go straight to storage with a signed URL.

```ts
// next.config.ts
const nextConfig: NextConfig = { serverActions: { bodySizeLimit: "4mb" } };
```

```bash
# is the 413 the framework or the platform? a plain route answers that
curl -s -o /dev/null -w "%{http_code}\n" -X POST https://app.example.com/api/upload -F "file=@big.zip"
```

**The return value crosses a serialisation boundary.** `useActionState` keeps the action's result in
React state and re-sends it with the RSC payload, so an `Error`, `Date`, `Map` or class instance
cannot make the trip; production replaces an uncaught throw with a digest and the user sees
`An error occurred in the Server Components render`. Return plain objects, convert timestamps to ISO
strings at the boundary.

```ts
type State = { ok: boolean; message: string; fieldErrors?: Record<string, string> };
```

| Boundary | Crosses when | Fails when |
|---|---|---|
| client → action arguments | `FormData`, serialisable values | a `File` outside `FormData` |
| action → client return | plain object, string, number, array | `Error`, `Date`, `Map`, class |
| action → framework | thrown control-flow errors | caught in `try/catch` |

Two more: `revalidatePath`/`revalidateTag` go **before** the redirect, because the redirect ends the
action; and an action used directly as `onClick={myAction}` posts nothing — wrap it in
`startTransition` or use `<form action={…}>`.

## 8. Order of investigation

1. Running locally but not in the image? `docker run --rm -it my-app sh -lc 'ls .next/static/css | head; ls public | head'` before editing any config.
2. Styled but icons missing → `public/`. Icon and static both present but 404 behind a proxy → `assetPrefix` or the proxy route.
3. Images 500 → `ls node_modules/@img`; empty means pruned, present but throwing means wrong libc.
4. Auth loop → log `hasCookie` and `expires_at` (section 6) before touching the proxy matcher. A matcher like `/dashboard/:path*` never runs for `/`, so a refresh there is discarded.
5. Form that does not navigate → look for `try {` above the `redirect(` before anything else.

The longer walkthroughs with worked examples:
[Next.js standalone in Docker: missing CSS and the `sharp` module](https://www.ryuslog.com/blog/nextjs-standalone-missing-static-css-and-sharp-docker-fixes) ·
[Supabase SSR cookies and a stale auth session](https://www.ryuslog.com/blog/nextjs-supabase-ssr-cookies-and-stale-auth-session-fixes) ·
[Server Actions `redirect()` in a `try/catch`](https://www.ryuslog.com/blog/nextjs-server-actions-redirect-try-catch-and-formdata-errors) ·
[Next.js 16 upgrade breaks](https://www.ryuslog.com/blog/nextjs-16-upgrade-breaking-changes-async-params-proxy-turbopack).

Browser tools that help here, all client-side:
[cURL → fetch/Python/Go](https://www.ryuslog.com/tools/curl) to check `/_next/static/…` and its
content type · [JWT decoder](https://www.ryuslog.com/tools/jwt) to read `exp`, `sub`, `role` out of
an `sb-…-auth-token` cookie · [diff](https://www.ryuslog.com/tools/diff) to compare two builds'
chunk lists · [timestamp converter](https://www.ryuslog.com/tools/timestamp) for `expires_at`.
