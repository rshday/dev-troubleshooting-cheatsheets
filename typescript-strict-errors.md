# TypeScript strict mode: the errors you actually hit

Turn `strict` on, get 400 errors, and the temptation is `as any`. These are the same dozen
errors, each with the fix that survives code review.

```jsonc
// tsconfig.json — the flags worth knowing individually
{
  "compilerOptions": {
    "strict": true,                    // noImplicitAny, strictNullChecks, strictFunctionTypes, ...
    "noUncheckedIndexedAccess": true,  // arr[0] is T | undefined
    "exactOptionalPropertyTypes": true,// { a?: string } rejects { a: undefined }
    "noImplicitOverride": true,
    "noFallthroughCasesInSwitch": true
  }
}
```

`noUncheckedIndexedAccess` and `exactOptionalPropertyTypes` are **not** part of `strict` — and
they are the two that surface real bugs, so most of the errors below come from them.

## Quick index

| Code | Message (short) | Section |
|---|---|---|
| TS7006 | Parameter implicitly has an `any` type | [1](#1-ts7006-implicit-any-in-a-parameter) |
| TS18046 / TS2571 | `catch` variable / value is of type `unknown` | [2](#2-unknown-is-not-a-type-you-can-call) |
| TS18048 / TS2532 | Value is possibly `undefined` | [3](#3-possibly-undefined-under-no-checked-indexed-access) |
| TS7053 | Element implicitly has an `any` type… can't index with `string` | [4](#4-ts7053-indexing-with-a-string) |
| TS2339 | Property does not exist on type | [5](#5-ts2339-property-does-not-exist-narrow-first) |
| TS2345 / TS2322 | Argument / type not assignable | [6](#6-assignability-errors-read-the-target-type) |
| TS2769 | No overload matches this call | [7](#7-ts2769-overload-resolution) |
| TS2564 | Property has no initializer and is not definitely assigned | [8](#8-ts2564-class-fields) |
| TS2416 | Property in type is not assignable to the same property in base type | [9](#9-ts2416-overrides-that-change-the-contract) |
| TS2589 | Type instantiation is excessively deep | [10](#10-ts2589-recursive-conditional-types) |

---

## 1. TS7006: implicit `any` in a parameter

The error names the parameter, not the cause — most of the time the callback's shape was never
declared.

```ts
// ✗ TS7006: Parameter 'row' implicitly has an 'any' type.
rows.map((row) => row.id);

// ✓ say what the data is, once
type Row = { id: string; qty: number };
const rows: Row[] = await loadRows();
rows.map((row) => row.id);

// ✓ when the source is external and genuinely unknown, parse it at the boundary
const parsed: unknown = JSON.parse(text);
function isRow(v: unknown): v is Row {
  return typeof v === 'object' && v !== null && 'id' in v && 'qty' in v;
}
if (!Array.isArray(parsed) || !parsed.every(isRow)) throw new Error('bad payload');
```

A `noImplicitAny` error inside a library callback usually means the generic was lost upstream —
fix the upstream type rather than annotating the callback.

## 2. `unknown` is not a type you can call

```ts
try {
  await fetchUser();
} catch (e) {
  // ✗ TS18046: 'e' is of type 'unknown'.
  console.log(e.message);
}

// ✓ narrow it — anything can be thrown, including a string
} catch (e) {
  if (e instanceof Error) console.error(e.message);
  else console.error(String(e));
}
```

`instanceof Error` fails across realms and for errors that were serialised; for HTTP clients that
throw plain objects, check a `code`/`status` property with a guard. Getting this distinction right
is the point of [unknown vs any](https://www.ryuslog.com/blog/typescript-unknown-vs-any-type-safe-dynamic-values).

## 3. "Possibly `undefined`" under no-checked-indexed-access

```ts
const first = items[0];          // T | undefined, even for a non-empty array literal's type
const map: Record<string, number> = {};
const v = map['a'];              // number | undefined
```

Three legitimate fixes, in order of preference:

```ts
// a) check — the array really can be empty
if (first === undefined) return null;

// b) destructure with a default when "missing" has a meaning
const [head = fallback] = items;

// c) assert once, where you have already proved it
if (items.length === 0) throw new Error('empty');
const definite = items[0]!;      // the non-null assertion is now honest
```

What not to do: `items[0] as T`. It compiles and removes the very check the flag exists for. A
`!` that is preceded by a real check is documentation; `as T` is a lie the compiler cannot audit.

## 4. TS7053: indexing with a string

```ts
const counts: Record<string, number> = {};
for (const key of Object.keys(cfg)) counts[key] = 1;   // fine with Record

const cfg = { a: 1, b: 2 };
cfg[key];        // ✗ TS7053: expression of type 'string' can't be used to index
```

```ts
// ✓ the key space is known
function read(k: 'a' | 'b') { return cfg[k]; }

// ✓ the key space is open
const open: Record<string, number> = cfg;

// ✓ you really want a lookup table, not an object
const byName = new Map<string, number>(Object.entries(cfg));
byName.get(key);
```

`Object.keys()` widens keys to `string` by design (objects can have more keys than the type
declares), so `Object.keys(cfg) as (keyof typeof cfg)[]` is acceptable where you control the
object's construction — but prefer `Object.entries()` and let the value keep its type.

## 5. TS2339: property does not exist — narrow first

```ts
type Shape = { kind: 'circle'; r: number } | { kind: 'square'; s: number };

function area(sh: Shape) {
  // ✗ TS2339: Property 'r' does not exist on type 'Shape'.
  return Math.PI * sh.r ** 2;
}

// ✓ a discriminated union makes narrowing free
function area2(sh: Shape) {
  switch (sh.kind) {
    case 'circle': return Math.PI * sh.r ** 2;
    case 'square': return sh.s ** 2;
    default: {
      const never: never = sh;   // exhaustiveness: a new member is a compile error here
      throw new Error(`unhandled ${JSON.stringify(never)}`);
    }
  }
}
```

If the property exists at runtime but not in the type, the type is wrong — add it, or model the
extra fields with an index signature. Casting the whole object to `any` hides the next three bugs too.
See [discriminated unions and exhaustiveness](https://www.ryuslog.com/blog/typescript-discriminated-unions-exhaustiveness-checking-never).

## 6. Assignability errors: read the target type

```
TS2322: Type 'string | undefined' is not assignable to type 'string'.
TS2345: Argument of type '{ id: string; }' is not assignable to parameter of type 'User'.
```

The error prints both sides. The useful question is *which* property or member is incompatible —
hover the type in the editor, or ask `tsc` for more:

```bash
npx tsc --noEmit --pretty false | grep -A3 'TS2322'
```

Common real causes:

| Cause | Fix |
|---|---|
| Optional vs required (`a?: string` passed where `a: string` is required) | provide a default at the boundary |
| `readonly` vs mutable array | `[...arr]` or type the parameter as `readonly T[]` |
| Excess property from an object literal | the target type genuinely lacks it — widen the target or pick the fields |
| Literal widened to `string` | `as const`, or type the target as `string` |

`exactOptionalPropertyTypes` adds one more: `{ a?: string }` no longer accepts `{ a: undefined }`.
Build optional fields conditionally (`...(a !== undefined && { a })`) rather than passing `undefined`.

## 7. TS2769: overload resolution

```
TS2769: No overload matches this call.
  Overload 1 of 3, '(input: string, init?: RequestInit): Promise<Response>', gave the following error.
```

Read the **first** "gave the following error" line — that is the overload whose shape came
closest, and the mismatch it names is usually the actual bug (a wrong option name, a `null` where
the signature wants `undefined`, a handler whose parameter type does not match the event).

```ts
// ✗ passing a value where the overload wants a narrowed literal
router.get('/x', (req, res) => {});
// ✓ give the handler the framework's own type
router.get('/x', (req: Request, res: Response) => {});
```

## 8. TS2564: class fields

```ts
class Client {
  private token: string;   // ✗ TS2564: no initializer and not definitely assigned
  constructor(t: string) { this.token = t; }   // assigned in the constructor → OK
}
```

The error appears when the assignment is in a method, in a conditional, or in a base class. Fixes:

```ts
private token!: string;                  // assigned by an init hook — use sparingly, documented
private token = '';                      // a real default
constructor(private readonly token: string) {}   // parameter property, cleanest
```

`!` on a field means "trust me, this is set before use". If an `init()` can be skipped, that trust
is a runtime `undefined`.

## 9. TS2416: overrides that change the contract

```ts
class Base { save(x: string | number): void {} }
class Child extends Base {
  // ✗ TS2416: parameter type narrows in the override
  override save(x: string): void {}
}
```

An override may accept **more** than the base, never less (Liskov). If the child genuinely needs a
narrower input, the base signature is wrong: make it generic
(`save<T extends string | number>(x: T)`), or split the method.

## 10. TS2589: recursive conditional types

```
TS2589: Type instantiation is excessively deep and possibly infinite.
```

Usually a mapped/conditional type applied to a self-referential structure. Practical fixes:

```ts
// a) break the recursion at a depth with a tuple counter
type Deep<T, D extends unknown[] = []> =
  D['length'] extends 5 ? unknown : { [K in keyof T]: Deep<T[K], [...D, 0]> };

// b) constrain the input (interfaces, not recursive type aliases)
// c) annotate the intermediate variable so the checker stops inferring
const x: Deep<Config> = value;
```

---

## The three habits that remove most strict errors

1. **`unknown` at the edge, typed inside.** `JSON.parse`, `fetch().json()`, `localStorage`, env vars — validate once with a guard, then the rest of the file has real types.
2. **`satisfies` instead of annotation** when you want both the literal's specificity and the shape's checking: `const routes = { home: '/' } satisfies Record<string, `/${string}`>` keeps `routes.home` as `'/'`.
3. **Narrowing functions return predicates.** `function isUser(v: unknown): v is User` turns a scattered set of casts into one testable function.

Further reading: [utility types: Pick, Omit, Record, Partial](https://www.ryuslog.com/blog/typescript-utility-types-pick-omit-record-partial) ·
[satisfies vs type annotations](https://www.ryuslog.com/blog/typescript-satisfies-operator-syntax-vs-type-annotations) ·
live checks: [JSON → TypeScript types](https://www.ryuslog.com/tools/json),
[regex tester](https://www.ryuslog.com/tools/regex)
