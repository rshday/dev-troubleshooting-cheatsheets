# Bash: writing scripts that fail loudly and delete nothing important

Every item here is a bug that shipped once. The pattern is the same: a command failed, the script
continued, and the failure surfaced three steps later as something unrecognisable.

## 1. The header

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
IFS=$'\n\t'
```

| Flag | Effect | Gotcha |
|---|---|---|
| `-e` | exit on a failing command | **not** triggered in `if`, `while`, `&&`, `||`, `!`, or a pipeline's non-final command |
| `-u` | error on an unset variable | `${var:-default}` and `${var+set}` are still safe |
| `-o pipefail` | a pipeline fails if any stage fails | bash only, not POSIX `sh` |
| `-E` | `ERR` trap inherited by functions and subshells | needed or your trap misses half the failures |
| `-x` | trace | never leave it on; it prints secrets |

`set -e` is not a sandbox. These still continue:

```bash
grep 'x' file || true                 # intended
if ! out=$(cmd); then ...; fi         # intended
cmd | tee log                         # tee succeeds; cmd's failure hidden without pipefail
local v=$(false)                      # `local` returns 0, the failure is lost  ← classic
```

That last one is worth memorising: `local x=$(failing)` swallows the error because `local`'s own
exit status is what `set -e` sees. Split it:

```bash
local x
x=$(failing) || return 1
```

## 2. Quote, always, and know the three levels

```bash
rm -- "$file"                   # word splitting off, globbing off, and -- for a leading -
cp "$src" "$dst"                # correct
cp $src $dst                    # a path with a space becomes two arguments
args=("${@}")                   # arrays for argument lists
for p in "${paths[@]}"; do ...; done
```

- `"$var"` — no splitting, no globbing.
- `$var` — split on `IFS`, globs expanded. This is what turns `rm -rf $dir` into a disaster when `dir` is empty and unset under `-u` off.
- `${var//pat/rep}` — pattern substitution, quoting changes the meaning of the pattern.

`[ ]` is `test`, `[[ ]]` is bash's own; only `[[ ]]` does `&&`/`||`, `=~`, and no-split comparison:

```bash
if [[ -n $x && $x == *.log ]]; then ...; fi
if [[ $name =~ ^[a-z0-9_-]+$ ]]; then ...; fi     # unquoted right side = regex
```

## 3. Deleting safely

```bash
: "${TARGET_DIR:?refusing to run without TARGET_DIR}"

case "$TARGET_DIR" in
  /|/home|/usr|/var) echo "refusing to delete $TARGET_DIR" >&2; exit 1 ;;
esac

[[ -d $TARGET_DIR ]] || { echo "not a directory: $TARGET_DIR" >&2; exit 1; }
rm -rf -- "${TARGET_DIR%/}/"
```

- `${VAR:?msg}` exits when unset **or empty** — the single most useful line in any destructive script.
- `--` ends option parsing so a file named `-rf` cannot become a flag.
- `"${dir%/}/"` prevents `rm -rf "$dir/"` from collapsing to `/` when `dir` is empty or `/`.
- Prefer `trash`/`mv` to a timestamped quarantine over `rm` in anything scheduled; a cron job has no undo.

For a delete with a pattern, print first:

```bash
find "$dir" -name '*.tmp' -mtime +7 -print   # then replace -print with -delete
```

## 4. Temporary files and cleanup

```bash
tmp=$(mktemp -d) || exit 1
cleanup() { local rc=$?; rm -rf -- "$tmp"; exit "$rc"; }
trap cleanup EXIT INT TERM
```

- `mktemp` respects `$TMPDIR`, is atomic, and is not predictable — `"/tmp/$$.lock"` is a symlink attack waiting to happen.
- `local rc=$?` inside the trap preserves the script's real exit code; without it, the script reports the exit status of `rm`.
- `trap ... EXIT` runs on normal exit **and** on `set -e` failure. Add `INT TERM` for containers: PID 1 in a container often ignores SIGTERM unless your script handles it.

## 5. Reading input correctly

```bash
while IFS= read -r line; do        # -r: no backslash interpretation
  printf '%s\n' "$line"
done < file                         # input redirection, not `cat file |`

# weird filenames
find . -type f -print0 | xargs -0 -r grep -l 'TODO'

# never
for line in $(cat file); do ...; done   # splits on spaces, eats blank lines, globs
```

`-r` matters: without it, `read` strips backslashes, so a Windows path or a regex in the file changes shape.
`xargs -r` skips the run entirely when the list is empty — otherwise `grep` waits on stdin.

## 6. Structure

```bash
main() {
  parse_args "$@"
  require curl jq systemctl
  do_work
}
require() { for c in "$@"; do command -v "$c" >/dev/null || { echo "missing: $c" >&2; exit 127; }; done; }

parse_args() {
  while (($#)); do
    case $1 in
      -n|--dry-run) DRY_RUN=1 ;;
      --dir) dir=${2:?--dir needs a value}; shift ;;
      --) shift; break ;;
      -h|--help) usage; exit 0 ;;
      *) echo "unknown option: $1" >&2; usage >&2; exit 2 ;;
    esac
    shift
  done
}

main "$@"
```

- `main "$@"` at the bottom means the whole file was parsed before anything ran — a syntax error on line 200 does not leave the machine half-configured.
- `case` over `getopts` when you need long options; `getopts` cannot do `--flag`.
- `(($#))` is the arithmetic form; `(( $# ))` also works, and both are clearer than `[ $# -gt 0 ]`.

## 7. Subshell traps

```bash
# ✗ cd affects the rest of the script, and a failure is a silent wrong directory
cd "$dir"
rm -rf ./build

# ✓ the cd is scoped
( cd "$dir" && rm -rf ./build )

# ✓ or fail loudly
cd "$dir" || exit 1
```

`cd` failing is one of the most common ways a script deletes or writes in the wrong place. Treat every
`cd` as a step that must be checked.

## 8. Before you commit

```bash
shellcheck -x -S style script.sh
bash -n script.sh          # parse only
bash -x script.sh --dry-run
```

ShellCheck catches unquoted expansions, `local x=$(...)`, `for ... in $(...)`, useless cats, and
`[ ]` regex misuse — most of this sheet, automatically.

---

Related: [redirection, pipes, exit codes and `set -euo`](https://www.ryuslog.com/blog/linux-bash-redirection-pipes-exit-codes-and-set-euo) ·
[arrays, associative maps and parameter expansion](https://www.ryuslog.com/blog/linux-bash-arrays-associative-maps-and-parameter-expansion) ·
[jq/yq for JSON and YAML in the shell](https://www.ryuslog.com/blog/linux-jq-yq-and-shell-json-yaml-cli-parsing) ·
Quick checks: [JSON formatter](https://www.ryuslog.com/tools/json), [YAML ↔ JSON](https://www.ryuslog.com/tools/yaml)
