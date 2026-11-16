# Developer Troubleshooting Cheatsheets

Short, field-tested notes for the failures that eat an afternoon: a pod that keeps getting
killed, a `tsc` wall of errors, a shell script that "worked on my machine", a cron job that
never runs, a regex that hangs the browser.

Each sheet is a decision table first and an explanation second — the goal is to get from the
symptom to the actual cause in a couple of minutes.

## Sheets

| Sheet | Covers |
|---|---|
| [kubernetes-oom-and-probes.md](kubernetes-oom-and-probes.md) | `OOMKilled`, exit 137, requests vs limits, QoS, probe timing, `CrashLoopBackOff` triage |
| [typescript-strict-errors.md](typescript-strict-errors.md) | The strict-mode error codes you actually hit, and the fix that is not `as any` |
| [linux-bash-safety.md](linux-bash-safety.md) | `set -euo pipefail`, quoting, traps, safe `rm`, `xargs`, reading files line by line |
| [cron-and-regex-patterns.md](cron-and-regex-patterns.md) | Cron syntax pitfalls and DST, plus regex patterns that backtrack or break |
## Browser tools used while debugging

All client-side, nothing is uploaded: [JSON formatter](https://www.ryuslog.com/tools/json) ·
[YAML ↔ JSON](https://www.ryuslog.com/tools/yaml) · [diff](https://www.ryuslog.com/tools/diff) ·
[regex tester](https://www.ryuslog.com/tools/regex) · [JWT decoder](https://www.ryuslog.com/tools/jwt) ·
[base64](https://www.ryuslog.com/tools/base64) · [URL encoder](https://www.ryuslog.com/tools/url) ·
[hash](https://www.ryuslog.com/tools/hash) · [UUID](https://www.ryuslog.com/tools/uuid) ·
[timestamp](https://www.ryuslog.com/tools/timestamp) · [cron parser](https://www.ryuslog.com/tools/cron) ·
[colour + contrast](https://www.ryuslog.com/tools/color)

Longer write-ups live at **[RYUSLOG DEV](https://www.ryuslog.com)**.

## Contributing

Corrections are welcome — open an issue with the command, the version, and what actually
happened. Concrete output beats "doesn't work".

## License

MIT.

