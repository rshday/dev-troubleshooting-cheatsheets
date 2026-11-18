# Next.js Server Actions: boundaries, redirects and body limits

A Server Action is a POST to the current route, RSC-encoded, awaited by the client, and its
return value must survive the same encoding. Almost every "my action does nothing" bug is one
of those three facts being ignored.

## Symptom → cause table

| Symptom | Actual cause | Fix |
|---|---|---|
| `redirect("/dashboard")` runs, browser stays on the form | `redirect()` throws a control-flow exception; a surrounding `try/catch` swallows it and returns state instead | call `redirect()` **after** the `try/catch`, or rethrow with `unstable_rethrow(error)` |
| `notFound()` / `unauthorized()` never navigate | same swallowed control-flow error | same |
| `413 Request Entity Too Large` or `Body exceeded 1mb limit` | 1 MB default cap per action body (the whole multipart payload, not per file) | set `serverActions: { bodySizeLimit: "4mb" }`, validate size before upload, or upload direct-to-storage |
| `useActionState` never updates, prod shows a digest | the action threw; an `Error`/`Date`/`Map` is not serialisable across the boundary | return a plain object: `{ ok: false, message, fieldErrors }` |
| `Only plain objects, and a few built-ins, can be passed to Client Components` | non-serialisable value in returned state | convert `Date` → ISO string, `Map` → object, `Error` → your own string |
| `formData.get is not a function` | action signature not `(prevState, formData)` for `useActionState` | match the signature |
| Navigation lost after an `await` | the action already returned | do the redirect last |
| `revalidatePath` has no effect | it ran after `redirect()` | invalidate first, then redirect |

## The pattern that works

```ts
// app/actions.ts
"use server";
import { redirect } from "next/navigation";

export async function createPost(prevState: State, formData: FormData): Promise<State> {
  const title = String(formData.get("title") ?? "");

  try {
    if (!title) throw new Error("Title is required");
    await db.post.create({ data: { title } });
  } catch (error) {
    return { ok: false, message: (error as Error).message }; // plain object only
  }

  redirect("/posts");   // outside the try: redirect throws by design
}
```

If the redirect genuinely belongs inside the guarded block, use the framework predicate:

```ts
import { unstable_rethrow } from "next/navigation";

try {
  await save(formData);
  redirect("/done");
} catch (error) {
  unstable_rethrow(error);              // rethrows redirect / notFound / forbidden
  return { ok: false, message: "Save failed" };
}
```

`isRedirectError` from `next/dist/client/components/redirect-error` also works but is an
internal path that moves between minors; `unstable_rethrow` is the documented wrapper.

## Version boundaries

| Next.js | Redirect helper | Body size key |
|---|---|---|
| 13.4–13.5 | `isRedirectError` (internal path) | not configurable |
| 14.x | `isRedirectError` (unstable path) | `experimental.serverActions.bodySizeLimit` |
| 15.x | `unstable_rethrow` added | `serverActions.bodySizeLimit` |
| 16.x | `unstable_rethrow` | `serverActions.bodySizeLimit` |

## Two limits, not one

`Body exceeded 1mb limit` is the framework; a bare `413` with no Next.js log line is the
platform. Find out which before changing config:

```bash
curl -s -o /dev/null -w "%{http_code}\n" -X POST https://app.example.com/api/upload \
  -F "file=@big.zip"
```

Vercel caps serverless request bodies at 4.5 MB; `bodySizeLimit` above that changes nothing.

## Checklist

- [ ] `redirect()` is the last statement, or `unstable_rethrow` is in the catch.
- [ ] Action return types are plain JSON-compatible objects.
- [ ] `bodySizeLimit` sits at the top level (`serverActions`), not under `experimental`, on 15+.
- [ ] File size is checked on the client before the request.
- [ ] `revalidatePath` / `revalidateTag` run before `redirect`.
- [ ] Actions used outside a form are wrapped in `startTransition`.
- [ ] Production errors are expected to arrive as digests — never rely on `error.message` reaching the browser.

Longer write-up: [Next.js Server Actions: swallowed redirects and body limits](https://www.ryuslog.com/blog/nextjs-server-actions-redirect-try-catch-and-formdata-errors)
