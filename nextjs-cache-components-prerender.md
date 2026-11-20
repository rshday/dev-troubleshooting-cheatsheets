# Next.js Cache Components: uncached data during prerender

`cacheComponents: true` turns "is this route static?" into a build-time check. Every route needs a
static shell; anything that reads the request must sit behind `<Suspense>`, in a cache function, or
behind an explicit blocking route. The build error names the route, not the line.

## Symptom → cause → answer

| Symptom | Cause | Answer |
|---|---|---|
| `Next.js encountered uncached or runtime data during prerendering` | `cookies()`, `headers()`, `params`, `searchParams`, `connection()` or an uncached DB call outside `<Suspense>` | `<Suspense>` around the smallest part that reads it |
| The same error on a `fetch`/ORM call | No cache boundary for a stable, expensive read | `"use cache"` + `cacheLife()` + `cacheTag()` |
| Route can never have a shell (auth'd dashboard, payment callback) | First byte always depends on the session | `export const instant = false` |
| `"use cache"` compiles but serves one user's data to everyone | A cache function is shared across users and cannot read a cookie | Read the cookie outside, cache by argument, tag per id |
| `revalidateTag` takes one argument → type error | Next 16 requires the cache profile | `revalidateTag(\`product:${id}\`, "max")` |
| `missing-suspense-with-csr-bailout` in a Client Component | Different failure: `useSearchParams()` in the client tree | Fix in the component tree with `<Suspense>`, not with `instant` |

## Minimal shapes

```tsx
// request-time read behind a boundary
export default async function Page({ params }: PageProps<"/products/[id]">) {
  const { id } = await params;
  return (
    <Suspense fallback={<p>Loading…</p>}>
      <ProductDetail id={id} />
    </Suspense>
  );
}
```

```ts
export async function getProduct(id: string) {
  "use cache";
  cacheLife("hours");
  cacheTag(`product:${id}`);
  return db.product.findUnique({ where: { id } });
}
```

```tsx
export const instant = false; // session-dependent, never cached
```

## Order of operations when a build fails

1. `grep -rn "cookies()\|headers()\|searchParams\|connection()" app/<segment>` — the offending read
   is often several components below the page.
2. Decide per read: stream it, cache it, or block the route.
3. Wrap the smallest part, not the page. A page-level `<Suspense>` builds but replaces the whole page
   with a spinner.
4. Re-run `next typegen` / `tsc --noEmit`: `PageProps<>` fails at the type level when props are not
   awaited, before the build reports a route.

## Version notes

- Node.js 20.9+ and Turbopack (default in Next 16) are prerequisites.
- `export const dynamic` no longer decides caching for these routes; `instant` and the cache
  boundaries do.
- Route handlers and `opengraph-image.tsx` reading their arguments need the same treatment.
- Keep `revalidateTag`'s tag identical on read and write, or invalidation silently does nothing.

Reference: <https://nextjs.org/docs/app/api-reference/config/next-config-js/cacheComponents>,
<https://nextjs.org/docs/app/api-reference/functions/revalidateTag>
