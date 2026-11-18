# asyncpg behind a transaction pooler, and FastAPI CORS that reports the wrong error

## 1. Symptom → cause

| Symptom | Most likely cause | Section |
|---|---|---|
| `asyncpg.exceptions.DuplicatePreparedStatementError: prepared statement "__asyncpg_stmt_1__" already exists` (SQLSTATE `42P05`), works locally | Client-side statement cache against **transaction** pooling (port 6543) | [2](#2-the-statement-cache-against-transaction-pooling) |
| `prepared statement "s0" already exists` from psycopg, node-postgres, JDBC or an ORM | Same cause, different driver's switch | [2](#2-the-statement-cache-against-transaction-pooling) |
| `prepared statement "__asyncpg_stmt_1__" does not exist` | The cache thinks it prepared on this server connection; it did not | [2](#2-the-statement-cache-against-transaction-pooling) |
| `ConnectionFailureError: connection is closed` mid-request, after a quiet period | The pooler closed an idle server connection; asyncpg learns on write | [3](#3-a-closed-connection-is-retryable-not-fatal) |
| `InterfaceError: cannot call … after close` | A closed connection object was reused | [3](#3-a-closed-connection-is-retryable-not-fatal) |
| `TooManyConnectionsError` under load only | App pool × workers exceeds the pooler's server budget | [4](#4-pool-sizing-when-a-pooler-is-already-in-front) |
| `socket.gaierror: [Errno -9] Address family for the host is not supported` | Supabase's direct host is IPv6-only in that project | [5](#5-supabase-host-and-user-names) |
| `The value of the 'Access-Control-Allow-Origin' header … must not be the wildcard '*' when the request's credentials mode is 'include'` | `allow_origins=["*"]` together with `allow_credentials=True` | [6](#6-cors-wildcard-plus-credentials) |
| CORS error in the console, but the network tab shows a **500** | `ServerErrorMiddleware` sits outside `CORSMiddleware` | [7](#7-the-500-that-looks-like-a-cors-bug) |
| Preflight 400 `Disallowed CORS method` / `Disallowed CORS headers` | `allow_methods` defaults to `["GET"]`; `Authorization` is not safelisted | [6](#6-cors-wildcard-plus-credentials) |
| Request works in `curl`, blocked in the browser with no `Allow-Origin` | Trailing slash or `www` mismatch — an origin is an exact string | [6](#6-cors-wildcard-plus-credentials) |
| Cross-site cookie silently missing after CORS is fixed | `SameSite=Lax` default; a cross-site XHR needs `None; Secure` | [8](#8-the-second-gate-cookie-samesite) |

## 2. The statement cache against transaction pooling

asyncpg keeps a client-side map from SQL text to a server-side prepared statement named
`__asyncpg_stmt_N__`, up to `statement_cache_size` (100 by default). Under a direct connection the
name is unique per session, so the cache is a pure win. Under **transaction** pooling each
transaction gets a different server connection, so two clients that both prepared
`__asyncpg_stmt_1__` can land on one server session and the second `prepare` collides with `42P05`.

```python
pool = await asyncpg.create_pool(
    dsn,                                   # …pooler.supabase.com:6543/postgres
    min_size=1,
    max_size=5,                            # small: the pooler multiplies, not the app
    statement_cache_size=0,                # no server-side prepared statements
    max_cached_statement_lifetime=0,
    command_timeout=30,
    server_settings={"application_name": "api"},
)
```

`create_async_engine` builds its own asyncpg pool, so SQLAlchemy needs the dialect-level switch as
well — there are two caches, and both must go:

```python
create_async_engine(
    "postgresql+asyncpg://user:pass@host:6543/postgres",
    poolclass=NullPool,
    connect_args={
        "statement_cache_size": 0,
        "prepared_statement_cache_size": 0,   # SQLAlchemy's own layer
    },
)
```

| Driver | Setting for transaction pooling |
|---|---|
| asyncpg | `statement_cache_size=0` |
| SQLAlchemy + asyncpg | `prepared_statement_cache_size=0` (plus the asyncpg one) |
| psycopg 3 | `prepare_threshold=None` |
| node-postgres | avoid the `name` option on `query()` / `prepare: false` |
| Django ORM | `DISABLE_SERVER_SIDE_CURSORS = True` |
| Rails ActiveRecord | `prepared_statements: false` |
| JDBC | `prepareThreshold=0` |

```python
# psycopg 3: stop the driver preparing at all
conn = psycopg.connect(dsn, prepare_threshold=None)
```

| Endpoint | Port | Prepared statements | App pool | Use for |
|---|---|---|---|---|
| Direct connection | 5432 | supported, cache them | your real budget | migrations, `LISTEN/NOTIFY`, admin |
| Session pooler | 5432 | supported per session | small | long-lived sessions, ORMs without pooling |
| Transaction pooler | 6543 | **not across transactions** | 1–5 per worker | most web traffic |
| Transaction pooler + cache | 6543 | collisions | any | never |

Do not "fix" this by catching `DuplicatePreparedStatementError` and retrying: the retry prepares the
same name on another server connection, so the error moves instead of disappearing. Conversation-level
state belongs on 5432 — `SET` (not `SET LOCAL`), advisory locks, `LISTEN`, temp tables, cursors and
`Alembic` DDL all need a session.

```python
async with pool.acquire() as conn:
    async with conn.transaction():
        await conn.execute("SET LOCAL statement_timeout = '5s'")   # survives the transaction only
        await conn.fetch("SELECT * FROM orders WHERE tenant_id = $1", tenant)
```

## 3. A closed connection is retryable, not fatal

The pooler recycles idle server connections well before a web worker notices. asyncpg keeps the
client object and only discovers the socket is gone when it writes, so the first request after a
quiet period fails and every later one succeeds.

```python
async def fetch_one(pool, sql, *args):
    for attempt in range(2):
        try:
            async with pool.acquire() as conn:
                return await conn.fetchrow(sql, *args)
        except (asyncpg.ConnectionFailureError, asyncpg.InterfaceError):
            if attempt:
                raise
```

| Failure | Raised by | Real cause | Action |
|---|---|---|---|
| `DuplicatePreparedStatementError` | asyncpg | cache against a transaction pooler | `statement_cache_size=0` |
| `ConnectionFailureError` | asyncpg | pooler closed an idle connection | retry once, shorten idle timeout |
| `InterfaceError: cannot call … after close` | asyncpg | closed connection object reused | acquire per request |
| `TooManyConnectionsError` | PostgreSQL | app pool larger than the pooler budget | shrink `max_size` |
| `InvalidPasswordError` | asyncpg | SCRAM vs md5, or wrong pooler user | `postgres.<project-ref>` |

Retry only on connection errors, never on a query error: replaying a failed `INSERT` is fine because
it did not commit, replaying one that timed out after commit is not. `pool_pre_ping` narrows the
window at checkout but does not remove the retry requirement. A 30-second `SELECT 1` keepalive with
`min_size >= 1` prevents the first-request failure after every quiet period without a query per
request.

## 4. Pool sizing when a pooler is already in front

The pooler multiplies connections across workers: four uvicorn workers with `max_size=20` demand 80
server connections from a pooler that may allow far fewer, and the error arrives only under load.

```python
max_size = 5     # per worker, not per app: 4 workers × 5 = 20 server connections at most
```

For serverless, `NullPool` (or `max_size=1`) with the pooler doing the real pooling is the simplest
correct configuration. Before raising any limit, check whether the pool is actually exhausted:

```sql
select state, count(*), max(now() - state_change) as oldest
from pg_stat_activity
where datname = current_database()
group by state order by 2 desc;
```

## 5. Supabase host and user names

- Direct host `db.<project-ref>.supabase.co` resolves to **IPv6 only** unless the IPv4 add-on is
  enabled. A container or CI runner without IPv6 reports the DNS-flavoured
  `[Errno -9] Address family for the host is not supported` — use a pooler host instead.
- The pooler username is `postgres.<project-ref>`, not `postgres`. The password is the **database**
  password, not the anon/service key.
- Poolers terminate TLS on a different host, so a copied `ssl="require"` can fail as
  `SSL SYSCALL error: EOF detected`.

## 6. CORS wildcard plus credentials

The middleware only rewrites headers; the browser decides, in two phases. The spec forbids
`Access-Control-Allow-Origin: *` together with `Access-Control-Allow-Credentials: true`, and older
Starlette builds chose the header by looking for a `Cookie` **request header** — which is not the
same as `credentials: "include"`. The first call of a session carries no cookie yet, receives `*`,
and is blocked. Newer Starlette echoes the request origin when credentials are allowed, which hides
the bug on a fresh install and leaves it on every pinned deployment.

```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://app.example.com", "http://localhost:3000"],   # never ["*"] with credentials
    allow_credentials=True,                  # only if cookies are actually used
    allow_methods=["GET", "POST", "PATCH", "DELETE", "OPTIONS"],
    allow_headers=["Content-Type", "Authorization", "X-Request-Id"],
    expose_headers=["X-Request-Id"],
    max_age=600,
)
```

| Configuration | Preflight | Credentialed fetch | Correct for |
|---|---|---|---|
| `["*"]`, credentials off | `*` | `*` | public read-only APIs |
| `["*"]`, credentials on | origin echoed on new Starlette | blocked on older builds | nothing |
| explicit list, credentials on | origin echoed | allowed | cookie or bearer auth |
| `allow_origin_regex=".*"`, credentials on | origin echoed | allowed | dynamic preview hosts |
| no middleware | no headers | blocked | never |

- An origin is an **exact string**: `https://app.example.com/` never matches
  `https://app.example.com`, and `www` is a different origin. A `null` origin (sandboxed iframe,
  `file://`, some Electron shells) cannot be allowed by name at all.
- `allow_methods` defaults to `["GET"]`, so the first `DELETE` fails at preflight with a 400.
- `Authorization` is not in the safelisted request headers; every bearer-token app must list it.
- `expose_headers` is what makes `response.headers.get("X-Request-Id")` non-`null` in `fetch`;
  DevTools showing the header is not the same thing.
- `max_age` is capped by Chromium at 7200 seconds; `-1` means no caching. Preflight is skipped
  entirely for "simple" requests (GET/POST with `application/x-www-form-urlencoded`,
  `multipart/form-data` or `text/plain`) — a JSON body forces one, which is why switching to
  `FormData` sometimes "fixes CORS".

Reproduce both phases without a browser:

```bash
# preflight: no cookies on it
curl -i -X OPTIONS https://api.example.com/me \
  -H "Origin: https://app.example.com" -H "Access-Control-Request-Method: GET" \
  | grep -i "access-control"

# simple request
curl -i https://api.example.com/me -H "Origin: https://app.example.com" | grep -i "access-control"
```

If the second response pairs `Access-Control-Allow-Origin: *` with
`Access-Control-Allow-Credentials: true`, a credentialed browser fetch discards it.

## 7. The 500 that looks like a CORS bug

`ServerErrorMiddleware` is the outermost ASGI layer, so an unhandled exception produces a plain
`Internal Server Error` before `CORSMiddleware` ever sees the response. The browser reports CORS; the
real error is only in the server log. `HTTPException`, `RequestValidationError` and `notFound` are
converted **inside** the routing layer, which is why `raise HTTPException(400)` looks fine and
`raise ValueError()` looks broken.

```python
ALLOWED = {"https://app.example.com", "http://localhost:3000"}

@app.exception_handler(Exception)
async def unhandled(request: Request, exc: Exception):
    logging.exception("unhandled", exc_info=exc)
    origin = request.headers.get("origin")
    headers = {"Access-Control-Allow-Origin": origin, "Vary": "Origin"} if origin in ALLOWED else {}
    return JSONResponse({"detail": "Internal Server Error"}, status_code=500, headers=headers)
```

Reading the same origin list in two places is ugly but honest: the error path runs outside the
middleware, so it has to know the same set.

## 8. The second gate: cookie `SameSite`

CORS and cookie policy are separate. A cross-site `fetch` with `credentials: "include"` sends the
cookie only if it was set with `SameSite=None; Secure`; `Lax`, the modern default, means the request
arrives authenticated as nobody.

```python
response.set_cookie("session", token, httponly=True, secure=True,
                    samesite="none", max_age=3600, path="/")
```

When the frontend and API ship together, a same-origin proxy (Next.js `rewrites`, NGINX
`proxy_pass`) removes the problem instead of configuring around it — but check that the ingress does
not strip `Origin`, or no CORS header is produced at all.

## 9. Order of investigation

1. `DuplicatePreparedStatement` or `does not exist` on a `.pooler.` / `:6543` host → driver cache, not the query. Fix the driver before reading the SQL.
2. Intermittent `ConnectionFailureError` → retry once; do not touch `max_size` yet.
3. `TooManyConnectionsError` only under load → `pg_stat_activity`, then pool size **per worker**.
4. CORS complaint → find the actual status code in the network tab first. A 500 is section 7, not section 6.
5. CORS fixed but the cookie is still missing → `SameSite`/`Secure` and the exact `Origin` string.

Full write-ups with worked examples:
[asyncpg `DuplicatePreparedStatement` on PgBouncer and Supabase](https://www.ryuslog.com/blog/python-asyncpg-connection-failure-and-pgbouncer-prepared-statements) ·
[FastAPI CORS: blocked preflight and the credentials wildcard](https://www.ryuslog.com/blog/python-fastapi-cors-preflight-blocked-and-credentials-wildcard-fix) ·
[PgBouncer transaction vs session pooling: what breaks](https://www.ryuslog.com/blog/postgres-pgbouncer-transaction-pooling-vs-session-pooling-errors).

Browser tools that help, all client-side:
[SQL formatter](https://www.ryuslog.com/tools/sql) to normalise the failing statement before diffing
its text · [cURL → fetch/Python/Go](https://www.ryuslog.com/tools/curl) to replay the preflight and
the simple request with the exact headers · [diff](https://www.ryuslog.com/tools/diff) for two
connection strings or two `EXPLAIN` outputs · [URL encoder](https://www.ryuslog.com/tools/url) to
split a `postgres://` DSN into host, port and parameters.
