# `@supabase/ssr` on Next.js: sessions that flip signed-out

One logical Supabase client is split into a server shape (cookie adapter) and a browser shape
(`document.cookie`). When the adapter is wrong the two disagree, and the session appears to
alternate between signed-in and signed-out on every refresh.

## Three failures, in the order they appear

| Layer | What breaks | Signal |
|---|---|---|
| Adapter | `cookies()` is async since Next.js 15, so a synchronous adapter reads an empty store | `getUser()` returns `null` on the first render |
| Write | A Server Component is a render; `cookieStore.set()` throws there and a `catch` discards the refreshed token | `Error: Cookies can only be modified in a Server Action or Route Handler` |
| Proxy | The refresh lands on the **response** while the route handler downstream reads the **request** | session works once, then dies; `NavigatorLockAcquireTimeoutError` in the console |

Fix all three together: return every chunk from `getAll`, write only where writing is legal,
and in the proxy copy each `setAll` cookie onto **both** the request and the response.

## Server client

```ts
// lib/supabase/server.ts — Next.js 15+/16, @supabase/ssr 0.5+
import { createServerClient } from "@supabase/ssr";
import { cookies } from "next/headers";

export async function createClient() {
  const cookieStore = await cookies();   // Promise since Next.js 15

  return createServerClient(url, anonKey, {
    cookies: {
      getAll: () => cookieStore.getAll(),  // every chunk, not just the base name
      setAll(cookiesToSet) {
        try {
          cookiesToSet.forEach(({ name, value, options }) =>
            cookieStore.set(name, value, options),
          );
        } catch {
          // render: illegal write. The proxy persists the refresh instead.
        }
      },
    },
  });
}
```

## Proxy (Next.js 16: `proxy.ts`)

```ts
export async function proxy(request: NextRequest) {
  let response = NextResponse.next({ request });

  const supabase = createServerClient(url, anonKey, {
    cookies: {
      getAll: () => request.cookies.getAll(),
      setAll(cookiesToSet) {
        cookiesToSet.forEach(({ name, value }) => request.cookies.set(name, value));
        response = NextResponse.next({ request });
        cookiesToSet.forEach(({ name, value, options }) =>
          response.cookies.set(name, value, options),
        );
      },
    },
  });

  const { data: { user } } = await supabase.auth.getUser();   // forces the refresh
  if (!user && request.nextUrl.pathname.startsWith("/dashboard")) {
    const url = request.nextUrl.clone();
    url.pathname = "/login";
    return NextResponse.redirect(url);
  }
  return response;
}
```

`request.cookies.set()` must run **before** `NextResponse.next({ request })` is rebuilt, and the
reassigned `response` is what you return.

## Adapter comparison

| Adapter | Works in a render | Persists the refresh | Typical failure |
|---|---|---|---|
| `getAll` / `setAll` | yes (writes skipped) | yes, via the proxy | none when the proxy returns its response |
| `get` / `set` / `remove` | partly | unreliable | drops `…-auth-token.1` chunks |
| omitted | reads `document` | never | `null` user on every request |
| two clients per request | yes | no | `NavigatorLockAcquireTimeoutError` |

Cookies over ~4 KB are chunked into `sb-<ref>-auth-token.0`, `.1`; an adapter that returns only
the base name hands back half a session.

## One log line that separates the three

```ts
const { data: { session } } = await supabase.auth.getSession();
console.log({
  hasCookie: (await cookies()).getAll().some(c => c.name.startsWith("sb-")),
  sessionExpiry: session?.expires_at,
  now: Math.floor(Date.now() / 1000),
});
```

- `hasCookie: false` → the adapter never saw the chunks.
- expired expiry with `hasCookie: true` → the refresh never persisted (proxy).
- null session, everything healthy → a client was created before the proxy ran.

## Version notes

| Item | Behaviour |
|---|---|
| `cookies()` | synchronous in 14; a Promise in 15 and 16 |
| Interception file | `middleware.ts` (edge) in 14/15; `proxy.ts` (Node only) in 16 |
| `middleware.ts` left on 16 | still runs, but a configured `runtime` fails the build |
| `supabase-js` < 2.107.0 | the navigator.locks mutex could deadlock when awaiting a Supabase call inside `onAuthStateChange`; keep that callback synchronous or upgrade |

## Checklist

- [ ] `cookies()` is awaited.
- [ ] `getAll` returns every chunk; `setAll` is present.
- [ ] The proxy sets request **and** response cookies.
- [ ] One client per request — a module-level singleton in the browser, never in a render body.
- [ ] Guards use `getUser()`, not `getSession()`.
- [ ] Server Actions that change auth call `revalidatePath`.
- [ ] The service-role key has no `NEXT_PUBLIC_` prefix.

Longer write-up: [Supabase SSR on Next.js: stale sessions and lock timeouts](https://www.ryuslog.com/blog/nextjs-supabase-ssr-cookies-and-stale-auth-session-fixes)
