# Postgres RLS recursion, pooling, and the checks that catch both

## 1. Symptom → cause

| Symptom | Most likely cause | Section |
|---|---|---|
| `ERROR: 42P17: infinite recursion detected in policy for relation "members"` | A policy on a table queries that same table | [2](#2-recursion-42p17) |
| Policy "works" as `postgres`, returns nothing for `authenticated` | The policy is fine; a `SELECT` policy is missing for `UPDATE ... RETURNING` | [3](#3-update-needs-a-select-policy) |
| Supabase returns `500` with `code: "42P17"` | Same as row 1, wrapped by PostgREST | [2](#2-recursion-42p17) |
| `remaining connection slots are reserved` / `too many clients already` | Pooler exhaustion, usually a direct connection string in a serverless function | [5](#5-pooling-in-serverless) |
| `prepared statement "s0" already exists` | Transaction-mode pooling with client-side prepared statements | [5](#5-pooling-in-serverless) |
| `permission denied for table x` after `security definer` | `search_path` pinned to a schema that does not hold the table, or execute revoked | [4](#4-definer-functions-fail-safe-checks) |

RLS adds a filter to **every** reference to a table — including references inside a policy. That one
sentence explains most of the errors below.

## 2. Recursion (42P17)

```sql
-- Broken: the policy asks members whether you are in members
create policy members_read on members for select
  using (
    exists (select 1 from members m
            where m.project_id = members.project_id and m.user_id = auth.uid())
  );
-- ERROR: 42P17: infinite recursion detected in policy for relation "members"
```

```sql
-- Fixed: one definer function reads the table as its owner, which bypasses RLS
create or replace function public.is_member(p_project uuid)
returns boolean
language sql
stable
security definer
set search_path = public
as $$
  select exists (
    select 1 from members m
    where m.project_id = p_project and m.user_id = (select auth.uid())
  );
$$;

revoke all on function public.is_member(uuid) from public;
grant execute on function public.is_member(uuid) to authenticated;

create policy members_read on members for select
  using (public.is_member(project_id));
```

| Approach | Breaks recursion | Watch out for |
|---|---|---|
| `SECURITY DEFINER` helper | Yes | Must pin `search_path` and revoke `public` execute |
| Denormalized owner column on the row | Yes | Has to stay in sync; no role granularity |
| View with `security_invoker = false` | Sometimes | Owner must bypass RLS; hides columns |
| Claims from `auth.jwt()` | Yes | Stale until the token refreshes |
| RLS disabled on the membership table | Yes | Every row visible to `anon` — never |

Two tables guarding each other (`projects` → `members` → `projects`) produce the same error with the
relation name of whichever table the planner entered second, which points at the wrong policy. Make
exactly one side bypass RLS.

## 3. `UPDATE` needs a `SELECT` policy

```sql
create policy members_update on members for update
  using (public.is_admin(project_id))
  with check (public.is_admin(project_id));
```

Without a `SELECT` policy, `update ... returning *` returns zero rows even though the update
succeeded. Not recursion, but the second most common "my policy does not work" report on the same
table. `FOR UPDATE` policies also need the `SELECT` policy to see the row at all.

| Policy command | Needs `WITH CHECK` | Recursion risk |
|---|---|---|
| `FOR SELECT` | No | High if it reads its own table |
| `FOR INSERT` | Yes (falls back to `USING`) | High in `WITH CHECK` |
| `FOR UPDATE` | Yes | High, plus needs a `SELECT` policy |
| `FOR DELETE` | No | High |
| `FOR ALL` | Yes | Highest — applies to every command |

## 4. Definer functions fail-safe checks

```sql
select proname, prosecdef, proconfig
from pg_proc where proname = 'is_member';
-- prosecdef must be true; proconfig should contain search_path=public
```

| Mistake | Result |
|---|---|
| Function without `security definer` | Runs as the caller, re-enters RLS, still recurses |
| `security definer` without `search_path` | Hijackable through a schema earlier in the path |
| Grantee left at `public` | Any role can call it with an arbitrary project id |
| `alter table ... force row level security` | Even the owner is filtered — a definer function owned by that owner recurses again |

With `FORCE`, the escape hatch is a separate owner role with `BYPASSRLS`
(`alter role app_owner bypassrls`), or dropping `FORCE` on the membership table.

Also make the check cheap: `(select auth.uid())` is hoisted to an `InitPlan` and evaluated once per
statement, while bare `auth.uid()` is a function call per row. Pair it with an index on
`(project_id, user_id)` or every policy check becomes a sequential scan.

## 5. Pooling in serverless

| Connection | When it breaks | Use instead |
|---|---|---|
| Direct `db.<ref>.supabase.co:5432` from a function | Each instance opens its own connection; ~60 concurrent invocations exhaust the pool | Pooler, transaction mode, port `6543` |
| Session-mode pooler (`5432` on the pooler host) | Holds a server connection for the session; fine for long-lived servers, not for functions | Transaction mode for serverless |
| Transaction mode + client prepared statements | `prepared statement "s0" already exists` — the statement is bound to a connection you no longer own | Disable prepared statements (`?pgbouncer=true`, `prepare: false` in node-postgres) |
| Migrations through the pooler | DDL inside an implicit transaction can land on another connection | `psql "$DIRECT_URL"` with the direct host |

```ts
// node-postgres against a transaction-mode pooler
const pool = new Pool({
  connectionString: process.env.DATABASE_URL_POOLED, // port 6543
  max: 1,            // one connection per function instance
  idleTimeoutMillis: 10_000,
  prepare: false,    // no named prepared statements
});
```

Rules that hold in every stack:

- Pool size is per instance, not per app. `max: 10` in a serverless function means 10 × invocations.
- Session state (`set role`, `set search_path`, temp tables) does not survive in transaction mode.
  Send it with the query, or use `set local` inside a transaction.
- A pooler is not a fix for a leak: check `pg_stat_activity` for `idle in transaction` before raising
  any limit.

```sql
select state, count(*), max(now() - state_change) as oldest
from pg_stat_activity
where datname = current_database()
group by state order by 2 desc;
```

## 6. Testing policies without breaking the app

```sql
begin;
set local role authenticated;
set local request.jwt.claims = '{"sub":"00000000-0000-0000-0000-000000000001","role":"authenticated"}';
select count(*) from members;
rollback;
```

`set local` scopes the change to the transaction, so a test session cannot leave the database in a
role it did not start with. On Supabase `auth.uid()` reads `request.jwt.claims`, which is why
setting the claim by hand reproduces a user's view without a real token.

## 7. Order of investigation

1. Reproduce as the real role (`set local role authenticated`), never as `postgres` — the owner
   bypasses RLS and hides the bug.
2. Read the relation name in 42P17 and find the policy whose expression mentions that relation.
3. Check the helper function's `prosecdef` and `proconfig` before rewriting the policy.
4. For empty results instead of errors, look for a missing `SELECT` policy.
5. For connection errors, count `pg_stat_activity` by state before touching pool sizes.

Full write-ups with worked examples:
[Postgres RLS 42P17: infinite recursion detected in policy](https://www.ryuslog.com/blog/postgres-row-level-security-infinite-recursion-policy-fix).

Browser tools that help while debugging, all client-side:
[SQL formatter](https://www.ryuslog.com/tools/sql) ·
[JSON formatter](https://www.ryuslog.com/tools/json) for PostgREST payloads ·
[UUID generator](https://www.ryuslog.com/tools/uuid) for test ids ·
[JWT decoder](https://www.ryuslog.com/tools/jwt) to read `sub` and `role` out of a Supabase token ·
[cURL → fetch/Python/Go](https://www.ryuslog.com/tools/curl) to replay a failing request.
