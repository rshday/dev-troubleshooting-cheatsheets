# Developer Troubleshooting Cheatsheets

Short, field-tested notes for the failures that eat an afternoon: a pod that keeps getting
killed, a `tsc` wall of errors, a shell script that "worked on my machine", a cron job that
never runs, a regex that hangs the browser, a Next.js upgrade that builds but 500s, a Postgres
policy that reports infinite recursion.

Each sheet is a decision table first and an explanation second — the goal is to get from the
symptom to the actual cause in a couple of minutes.

## Sheets

| Sheet | Covers |
|---|---|
| [kubernetes-oom-and-probes.md](kubernetes-oom-and-probes.md) | `OOMKilled`, exit 137, requests vs limits, QoS, probe timing, `CrashLoopBackOff` triage |
| [typescript-strict-errors.md](typescript-strict-errors.md) | The strict-mode error codes you actually hit, and the fix that is not `as any` |
| [linux-bash-safety.md](linux-bash-safety.md) | `set -euo pipefail`, quoting, traps, safe `rm`, `xargs`, reading files line by line |
| [cron-and-regex-patterns.md](cron-and-regex-patterns.md) | Cron syntax pitfalls and DST, plus regex patterns that backtrack or break |
| [nextjs-16-migration-gotchas.md](nextjs-16-migration-gotchas.md) | The five Next.js 16 breaks: async `params`, Turbopack vs a `webpack` config, `middleware.ts` → `proxy.ts`, missing `default.js`, Cache Components |
| [postgres-rls-and-pooling.md](postgres-rls-and-pooling.md) | `42P17` infinite recursion in policies, `security definer` checks, `UPDATE ... RETURNING` returning nothing, pooler exhaustion and prepared statements |
| [nextjs-docker-standalone-and-ssr.md](nextjs-docker-standalone-and-ssr.md) | `output: "standalone"` images missing `.next/static`, `public/` and the `sharp` binary; `@supabase/ssr` cookie adapters and a session that flips signed-out; `redirect()` swallowed by a `try/catch`, the 1 MB action body limit |
| [python-asyncpg-pgbouncer-and-fastapi-cors.md](python-asyncpg-pgbouncer-and-fastapi-cors.md) | `DuplicatePreparedStatement` on a PgBouncer/Supabase transaction pooler, retryable pooler disconnects, pool sizing; FastAPI `CORSMiddleware` wildcard-with-credentials, the 500 that looks like CORS, `SameSite` |
| [nextjs-server-actions-boundaries.md](nextjs-server-actions-boundaries.md) | `redirect()` swallowed by `try/catch`, what can cross the action boundary, the 1 MB body limit vs a platform `413`, `useActionState` returning digests |
| [nextjs-supabase-ssr-session.md](nextjs-supabase-ssr-session.md) | `@supabase/ssr` sessions flipping signed-out: async `cookies()`, illegal writes in a render, proxy refresh lost, `NavigatorLockAcquireTimeoutError` |
| [python-pip-pep668-and-venvs.md](python-pip-pep668-and-venvs.md) | `error: externally-managed-environment` (PEP 668): the marker file, venv vs pipx vs `apt`, distro Docker images, CI runners, `--user` being patched off, repairing a broken system Python |
| [nodejs-npm-eresolve-and-peer-deps.md](nodejs-npm-eresolve-and-peer-deps.md) | `ERESOLVE unable to resolve dependency tree`: reading `peer … from …`, `overrides` and `$package`, `--legacy-peer-deps` vs `--force`, workspace roots, stale lockfiles |
| [docker-daemon-socket-and-contexts.md](docker-daemon-socket-and-contexts.md) | `Cannot connect to the Docker daemon at unix:///var/run/docker.sock`: endpoint resolution order, socket inspection, systemd startup errors, group permissions, containers, Docker Desktop/WSL, rootless, SSH and TLS remotes |
| [nginx-502-upstream-errors.md](nginx-502-upstream-errors.md) | `502 Bad Gateway` triage from the error log: `111` vs `110`, premature close, `no live upstreams`, header buffers, `localhost` across containers, health checks and resolvers, PHP-FPM |
| [nextjs-cache-components-prerender.md](nextjs-cache-components-prerender.md) | `cacheComponents: true` and the `uncached or runtime data during prerendering` build error: `<Suspense>` placement, `"use cache"` with `cacheLife`/`cacheTag`, `instant = false`, the `revalidateTag` profile argument, Node 20.9/Turbopack prerequisites |
| [kubernetes-cgroup-v2-migration.md](kubernetes-cgroup-v2-migration.md) | Kubernetes 1.35 removing cgroup v1: a node that never registers, `failCgroupV1: false` as a dated escape hatch, the removed `--pod-infra-container-image` flag, containerd 1.x's last release, ipvs deprecation, upgrade order |

## Browser tools used while debugging

All client-side, nothing is uploaded: [JSON formatter](https://www.ryuslog.com/tools/json) ·
[YAML ↔ JSON](https://www.ryuslog.com/tools/yaml) · [diff](https://www.ryuslog.com/tools/diff) ·
[regex tester](https://www.ryuslog.com/tools/regex) · [JWT decoder](https://www.ryuslog.com/tools/jwt) ·
[base64](https://www.ryuslog.com/tools/base64) · [URL encoder](https://www.ryuslog.com/tools/url) ·
[hash](https://www.ryuslog.com/tools/hash) · [UUID](https://www.ryuslog.com/tools/uuid) ·
[timestamp](https://www.ryuslog.com/tools/timestamp) · [cron parser](https://www.ryuslog.com/tools/cron) ·
[colour + contrast](https://www.ryuslog.com/tools/color) ·
[SQL formatter](https://www.ryuslog.com/tools/sql) ·
[CSV ↔ JSON](https://www.ryuslog.com/tools/csv-json) ·
[cURL → fetch / Python / Go](https://www.ryuslog.com/tools/curl)

Longer write-ups live at **[RYUSLOG DEV](https://www.ryuslog.com)**.

## Contributing

Corrections are welcome — open an issue with the command, the version, and what actually
happened. Concrete output beats "doesn't work".

## License

MIT.

