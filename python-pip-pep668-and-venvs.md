# pip, PEP 668 and the "externally-managed-environment" wall

Every `pip install` that dies with `error: externally-managed-environment` is answering one
question: *who owns the files this install would overwrite?* The interpreter belongs to the
operating system, so the distribution marked it off-limits. Nothing is corrupted and no network
call failed.

```
error: externally-managed-environment
× This environment is externally managed
╰─> To install Python packages system-wide, try apt install python3-xyz ...
note: ... You can override this, at the risk of breaking your Python or OS installation,
by passing --break-system-packages.
```

## Decide first: which of these are you installing?

| What you are installing | Right home | Command |
| --- | --- | --- |
| An application or service with a dependency file | A project virtualenv | `python3 -m venv .venv && . .venv/bin/activate` |
| The same, faster resolver | uv | `uv venv && uv pip install -r requirements.txt` |
| A command-line tool (`black`, `poetry`, `httpie`, `ansible`) | pipx, isolated per tool | `pipx install black` |
| A library that another system program imports | The distribution | `apt install python3-requests` |
| A throwaway container layer you control | Explicit override | `pip install --break-system-packages ...` |
| Nothing — you just want the error gone | Wrong question | Delete nothing, read below |

The `note:` line is the escape hatch, not the instruction. The `╰─>` block above it names the
supported paths.

## What the marker actually is

PEP 668 defines a file. pip 23.0.1+ resolves the standard-library directory of the *running*
interpreter and refuses to install if `EXTERNALLY-MANAGED` is there:

```bash
python3 - <<'PY'
import os, sysconfig
marker = os.path.join(sysconfig.get_path("stdlib"), "EXTERNALLY-MANAGED")
print(sys.executable, marker, os.path.exists(marker))
if os.path.exists(marker):
    print(open(marker).read()[:300])
PY
```

Three properties decide most arguments about it:

- **It keys on the interpreter, not on privileges.** `sudo pip install` fails identically, and so
  does pip running as root inside a `debian:12` or `ubuntu:24.04` container.
- **It only guards that interpreter's default environment.** A venv created by the same
  interpreter is a different environment and installs normally.
- **The wording varies, the failure does not.** Debian, Fedora and Homebrew customise the `Error=`
  text, so two machines can print different sentences for the same refusal.

## Who ships the marker

| Platform | Marker | The message points to |
| --- | --- | --- |
| Debian 12 (Python 3.11) | yes | `apt install python3-xyz` |
| Ubuntu 23.04+ / 24.04 LTS | yes | `python3 -m venv`, `apt install python3-full` |
| Fedora 38+ | yes | `dnf install python3-xyz` |
| Linux Mint 21.3+ | yes | same as Ubuntu |
| Homebrew `python@3.12` | yes | `pipx` or a venv |
| python.org installer | never | — |
| `python:3.12-slim` (Docker Hub) | never (CPython built from source) | — |

That last row explains the classic confusion: the identical `pip install` works in
`python:3.12-slim` and fails on `ubuntu:24.04`.

## Escape routes ranked

| Approach | Who owns the files | Survives OS upgrades | Take it when |
| --- | --- | --- | --- |
| `python3 -m venv .venv` | you, per project | yes | default answer |
| `uv venv` / `uv add` | you, per project | yes | you want a lockfile and speed |
| `pipx install <tool>` | pipx, per tool | yes | CLI tools |
| `apt install python3-xyz` | the distribution | yes | system programs import it |
| `pip install --user` | `~/.local` | usually | legacy only — often still blocked |
| `--break-system-packages` | the distribution | no | one image layer you control |
| deleting `EXTERNALLY-MANAGED` | the distribution | no | never |

## Docker

A distro base needs a venv inside the image:

```dockerfile
FROM ubuntu:24.04
RUN apt-get update && apt-get install -y --no-install-recommends \
        python3 python3-venv ca-certificates && rm -rf /var/lib/apt/lists/*
RUN python3 -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"
RUN pip install --no-cache-dir -r requirements.txt
CMD ["python", "-m", "app"]
```

When an image genuinely must install into the system interpreter (a vendor installer that calls
`pip` itself), make the override visible and explain it in the file:

```dockerfile
ENV PIP_BREAK_SYSTEM_PACKAGES=1
RUN pip install --no-cache-dir -r requirements.txt
```

An override with no comment is what breaks at the next base-image rebuild. A cached layer also
hides a newly added marker: the `RUN pip install` re-runs unchanged and fails with no diff on your
side, so pin the base image digest when a build must reproduce.

## CI that passes locally and fails on the runner

`ubuntu-latest` ships a distro Python; a macOS python.org install does not. Do not trust whichever
`pip` resolves — name the interpreter:

```yaml
- uses: actions/setup-python@v5
  with:
    python-version: "3.12"
- run: python -m venv .venv
- run: .venv/bin/pip install -r requirements.txt
- run: .venv/bin/pytest
```

## `--user` is not the escape it used to be

Debian and Ubuntu patch the user site off, and the marker check runs first anyway:

```bash
python3 -m site --user-site
python3 -c "import site; print(site.ENABLE_USER_SITE)"   # False on Debian/Ubuntu
```

With `ENABLE_USER_SITE` false, `--user` produces a second, more confusing error instead of a fix.

## Someone already broke the system Python

Do not delete directories by hand. Ask pip what the distribution owns, then reinstall exactly
those files:

```bash
python3 -m pip list --path /usr/lib/python3/dist-packages --format=freeze
sudo apt-get install --reinstall python3 python3-apt python3-yaml
```

## Checks that save the afternoon

```bash
which python3 pip                      # two different Pythons is the usual bug
python3 -m pip --version               # confirm the pip you think you are using
python3 -c "import sys; print(sys.prefix != sys.base_prefix)"   # True inside a venv
python3 -m venv .venv || echo "install python3-venv / python3-full first"
```

- `python3 -m venv` can fail with `ensurepip is not available` **before pip is involved** — install
  `python3-venv` (Ubuntu calls the metapackage `python3-full`).
- A notebook that cannot import a package pip just installed is a kernel path problem:
  `python3 -m ipykernel install --user --name venv-312` and select that kernel.
- `PIP_REQUIRE_VIRTUALENV=true` produces "Could not find an activated virtual environment", which
  reads like the marker error but is your own guard.
- Homebrew: `brew install pipx` then `pipx install <tool>`; mixing Apple's and Homebrew's
  frameworks through `--user` is the failure mode to avoid.

## Related

- Full write-up: [pip error: externally-managed-environment (PEP 668) fix](https://www.ryuslog.com/blog/pip-externally-managed-environment-pep-668-venv-fix)
- [`.env` ↔ K8s / Compose parser](https://www.ryuslog.com/tools/env) when the difference between
  two machines is one environment variable
