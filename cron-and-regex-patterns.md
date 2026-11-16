# Cron syntax and regex patterns: the two mini-languages that fail quietly

## Part 1 — Cron

### The five fields

```
┌───────────── minute        (0-59)
│ ┌─────────── hour          (0-23)
│ │ ┌───────── day of month  (1-31)
│ │ │ ┌─────── month         (1-12, or JAN-DEC)
│ │ │ │ ┌───── day of week   (0-7, 0 and 7 are both Sunday)
│ │ │ │ │
* * * * *  command
```

| Expression | Runs |
|---|---|
| `*/5 * * * *` | every 5 minutes |
| `0 * * * *` | at minute 0 of every hour |
| `0 3 * * *` | 03:00 daily |
| `30 9 * * 1-5` | 09:30 on weekdays |
| `0 0 1 * *` | midnight on the 1st of each month |
| `0 4 * * 0` | 04:00 Sundays |
| `15,45 8-18 * * *` | :15 and :45 from 08:00 to 18:59 |
| `@reboot` | once, when the cron daemon starts |
| `@daily` `@hourly` `@weekly` `@monthly` | shorthand (not portable to every cron) |

**Day of month and day of week combine with OR, not AND.** `0 0 13 * 5` runs on the 13th **and**
on every Friday — not "Friday the 13th". To get that, use one field:

```bash
0 0 13 * *   # the 13th
```
and then test the weekday inside the script (`date +%u`).

### The gotchas that make a job "never run"

| Symptom | Cause |
|---|---|
| Works by hand, never from cron | cron's `PATH` is short (`/usr/bin:/bin`): use absolute paths or set `PATH=` at the top of the crontab |
| Nothing in the output, job may or may not have run | no mail agent → output is discarded; redirect: `>> /var/log/job.log 2>&1` |
| Command truncated at a `%` | `%` is a newline in crontab — escape it as `\%`, or put the command in a script |
| Environment variables missing | cron does not read `~/.bashrc`; set them in the crontab or source a file in the script |
| `~` not expanded | cron uses `/bin/sh` semantics in some implementations; write `$HOME` |
| Every-5-minutes *sometimes* skips | the job runs longer than the interval and overlaps → lock it |
| `/etc/cron.d` entry ignored | the system format has an extra **user** field: `* * * * * root /usr/local/bin/job` |
| Runs at the wrong hour after a DST change | fixed-offset cron has no idea about local time; the skipped hour simply does not fire |

### Locking a job so runs never overlap

```bash
#!/usr/bin/env bash
set -euo pipefail
exec 9>/var/lock/backup.lock
flock -n 9 || { echo "already running" >&2; exit 0; }
# ... work ...
```

`flock -n` exits immediately when another instance holds the lock. Without it, a job that takes
7 minutes on a 5-minute schedule will eventually have three copies fighting over the same files.

### Logging what actually happened

```bash
* * * * * /usr/local/bin/job.sh >> /var/log/job.log 2>&1
```

A cron job with no output and no exit-code handling is unobservable. At minimum log the timestamp
and exit code:

```bash
ts=$(date -Is); if /usr/local/bin/job.sh; then echo "$ts ok"; else echo "$ts FAILED rc=$?"; fi >> /var/log/job.log 2>&1
```

### Cron or a systemd timer?

| Need | Use |
|---|---|
| Run once at a wall-clock time, simple, one server | cron |
| Must run after downtime (`Persistent=true`) | systemd timer |
| Randomised delay, or "run every 10 min after boot" | systemd timer (`RandomizedDelaySec`, `OnBootSec`) |
| Per-run logs in the journal, dependencies, cgroup limits | systemd timer |
| Portable across containers/BSD/macOS | cron |

```ini
# /etc/systemd/system/backup.timer
[Timer]
OnCalendar=*-*-* 03:00:00
Persistent=true
RandomizedDelaySec=300

[Install]
WantedBy=timers.target
```

`systemctl list-timers` shows the next elapse time — the single most useful debugging command for
scheduled work, and the reason many people migrate.

Check an expression, including the next run times, before you install it:
**[cron parser](https://www.ryuslog.com/tools/cron)**. Background on crontab fields, `%` escaping
and systemd equivalents: [linux cron jobs, crontab syntax and systemd timers](https://www.ryuslog.com/blog/linux-cron-jobs-crontab-syntax-and-systemd-timers).

---

## Part 2 — Regular expressions

### Test the pattern, and the input, not just the happy path

**[regex tester](https://www.ryuslog.com/tools/regex)** shows capture groups and match positions.
The habit that matters: paste the *real* worst-case input, including the empty string and a
1,000-character line.

### Greedy, lazy, and what "backtracking" costs

```text
".*"      greedy  — matches to the last quote on the line
".*?"     lazy    — matches to the first quote
```

Greedy vs lazy is correct in both cases; the failure mode is when a pattern can match many ways and
the engine tries them all:

```text
^(a+)+$        ← catastrophic backtracking on "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaa!"
^(\w+\s?)+$    ← same shape: nested quantifier over an overlapping class
```

Time to failure grows exponentially with input length. Two rules:

1. **Never nest a quantifier inside another quantifier over overlapping character classes.**
2. **Anchor and constrain.** `^\d{4}-\d{2}-\d{2}$` cannot backtrack badly; `.*\d.*\d.*` can.

JavaScript has no atomic groups or possessive quantifiers (`(?>…)`, `a++`) — so the fixes are
structural: rewrite as a single bounded quantifier, or validate in steps (split, then test each
part). RE2-based engines (Go, Rust `regex`, RE2, Cloudflare) have no backtracking at all and will
simply reject many of these patterns — that refusal is a feature.

### JavaScript-specific traps

```js
const re = /foo/g;
re.test('foo');   // true
re.test('foo');   // false  ← lastIndex advanced, and never reset
```

A `/g` (or `/y`) regex is stateful. Use it with `.matchAll()`, reset `.lastIndex = 0`, or drop the flag.

```js
'a.b'.replace('.', '-');          // 'a-b' is wrong; '.' matches anything
'a.b'.replace(/\./g, '-');        // 'a-b' correct
new RegExp('\\d+')                // in a string, every backslash doubles
/a.b/s.test('a\nb')               // dotAll: . matches newlines
/\bword\b/u                       // u: unicode-aware; \b alone is ASCII-only even with /u
str.replaceAll(/x/g, 'y')         // replaceAll needs /g, or it throws
```

| Need | Pattern |
|---|---|
| Any character including newlines | `[\s\S]`, or `.` with the `s` flag |
| Start/end of each line | `m` flag with `^`/`$` |
| Named capture | `(?<year>\d{4})` → `m.groups.year` |
| Lookahead / lookbehind | `(?=…)`, `(?<=…)` — lookbehind is supported in all current engines, but not older Safari |
| Literal backslash in a string-built regex | `new RegExp('\\\\')` |
| Escaping user input | `input.replace(/[.*+?^${}()|[\]\\]/g, '\\$&')` |

### A validation set worth copying

```js
const ISO_DATE = /^\d{4}-(0[1-9]|1[0-2])-(0[1-9]|[12]\d|3[01])$/;
const SEMVER   = /^(0|[1-9]\d*)\.(0|[1-9]\d*)\.(0|[1-9]\d*)(?:-[0-9A-Za-z.-]+)?(?:\+[0-9A-Za-z.-]+)?$/;
const EMAIL_OK = /^[^\s@]+@[^\s@]+\.[^\s@]{2,}$/;      // deliberately loose: RFC 5322 regex is a trap
const UUID_V4  = /^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/i;
const HEX_COLOR= /^#(?:[0-9a-f]{3}|[0-9a-f]{6})$/i;
```

`EMAIL_OK` is loose **on purpose**: a strict RFC-compliant pattern rejects valid addresses and
still accepts some invalid ones. Send a confirmation mail instead of trying to out-parse the RFC.

---

Related: [Bash: redirection, pipes and exit codes](https://www.ryuslog.com/blog/linux-bash-redirection-pipes-exit-codes-and-set-euo) ·
[regex tester](https://www.ryuslog.com/tools/regex) · [cron parser](https://www.ryuslog.com/tools/cron) ·
[timestamp converter](https://www.ryuslog.com/tools/timestamp) for checking what a scheduled time
actually maps to.
