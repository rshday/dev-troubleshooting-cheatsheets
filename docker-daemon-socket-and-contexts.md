# Cannot connect to the Docker daemon at unix:///var/run/docker.sock

Every Docker client command fails before it reaches a container:

```
$ docker ps
Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?
```

The sentence names the endpoint the client tried and nothing else. Four unrelated situations
produce it word for word: the daemon is not running, the client resolved a different endpoint,
your user has no permission on the socket, or the client lives inside a container that never
mounted the socket. Check them in that order — a careless fix for one makes the others look
worse.

## 1. Which endpoint did the client actually resolve?

The client is a thin RPC shell. The endpoint wins in this order, last match taking it:

| Order | Source |
| --- | --- |
| 1 | `--host` / `-H` on the command |
| 2 | `DOCKER_HOST` environment variable |
| 3 | `DOCKER_CONTEXT` environment variable |
| 4 | `currentContext` in `~/.docker/config.json` |
| 5 | built-in `default`: `unix:///var/run/docker.sock` |

```bash
env | grep -E '^DOCKER_(HOST|CONTEXT|TLS|CERT_PATH)'
docker context ls
docker context show
```

An asterisk beside `desktop-linux` on a server, or a `DOCKER_HOST` exported in a dotfile for a
host you stopped using, is the whole bug:

```bash
unset DOCKER_HOST
docker context use default
```

The error message prints `/var/run/docker.sock` even when `DOCKER_HOST` points somewhere else, so
never trust its path over `docker context show`.

## 2. Is anything listening there?

The socket is an ordinary filesystem entry, so inspect it without Docker:

```bash
ls -l /var/run/docker.sock          # srw-rw---- root docker
ss -xlp | grep docker.sock          # a dockerd line, or nothing
```

| Observation | Meaning | Next |
| --- | --- | --- |
| `No such file` | the daemon never created a socket | start the daemon, read its log |
| file exists, no `dockerd` in `ss` | nothing listening | start or restart the daemon |
| `srw-rw----`, `dockerd` listening | connectivity is fine | it is a permission problem |

## 3. Start the daemon and read its complaint

```bash
sudo systemctl enable --now docker
sudo systemctl status docker
sudo journalctl -u docker -n 50 --no-pager
```

| Startup line | Cause |
| --- | --- |
| `error initializing graphdriver` | data root on an unsupported filesystem, often an overlay mount inside a container |
| `pid file found, ensure docker is not running` | stale `docker.pid` after a hard reboot; the next start normally clears it |
| `failed to start daemon: ... address already in use` | another `dockerd` or something on the same socket path |

On Alpine, in a container image, or in WSL with systemd disabled, `systemctl` fails on its own.
Run `sudo dockerd --debug` and read the output.

## 4. Permission on the socket

`sudo docker ps` working while plain `docker ps` fails means the daemon is healthy and the group
membership is not.

```bash
sudo usermod -aG docker "$USER"
newgrp docker          # or log out and back in
id -nG                 # must now list docker
ls -l /var/run/docker.sock
```

Group membership is applied at login, so the current shell keeps the old list until a new session.
`sudo` proves the daemon is alive; it never fixes the permission.

## 5. Inside a container

An image with only the CLI reproduces the error exactly, because the socket is a host path:

```bash
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -u "$(id -u):$(id -g)" \
  docker:cli ps
```

```yaml
services:
  runner:
    image: docker:cli
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
```

Two traps. The container user must be allowed to open the host socket (hence the UID flag), and
mounting the socket grants the container root-equivalent control of the host daemon — do it only
for trusted build or CI containers. A `RUN` step in a Dockerfile cannot use the host socket at
all; use BuildKit's `--mount=type=ssh` for remote access instead.

## 6. Docker Desktop and WSL

Desktop uses its own context and a user-profile socket instead of `/var/run/docker.sock`.

| State | Symptom | Fix |
| --- | --- | --- |
| Desktop app closed | CLI cannot reach the engine | open the app, wait for "Engine running" |
| WSL integration off | `docker` shim missing in the distro | enable it for that distro, reopen the shell |
| leftover `DOCKER_HOST` | Desktop context ignored | `unset DOCKER_HOST`, then `docker context use desktop-linux` |
| no systemd in WSL | `systemctl start docker` → "System has not been booted with systemd" | enable systemd in `/etc/wsl.conf`, or start `dockerd` manually |

## 7. Rootless Docker listens elsewhere

Rootless mode runs the daemon as your user on `$XDG_RUNTIME_DIR/docker.sock`. Cron entries and
systemd units start without that variable and fall back to the default context:

```bash
export XDG_RUNTIME_DIR="/run/user/$(id -u)"
export DOCKER_HOST="unix://$XDG_RUNTIME_DIR/docker.sock"
systemctl --user status docker
```

## 8. Remote daemons (SSH, TLS)

The same message appears when SSH cannot reach the host or rejects the key. Ask the client to be
verbose:

```bash
docker -D -H ssh://deploy-account-on-remote ps
```

`-D` prints the SSH command it builds; if that command works on its own, the wire is fine. For
TLS, `DOCKER_HOST`, `DOCKER_TLS_VERIFY=1` and `DOCKER_CERT_PATH` must all be set — a missing one
gives this generic error rather than a handshake error.

## Mistakes that keep the message alive

- **Adding the user to `docker` and expecting the open shell to change.** It applies to new sessions.
- **Changing a context and assuming it is shell-local.** It is written to `~/.docker/config.json`
  and applies to every shell for that user, CI included.
- **Exporting `DOCKER_HOST` to test a remote host once.** The variable outranks contexts and
  survives the reason you set it; keep remotes as named contexts.
- **Confusing this with an API version mismatch.** "client version is too new" is a different
  failure with a different fix.
- **Checking `docker info` only.** `docker version` splits client and server, which is the fastest
  way to prove whether the binary or the connection is broken.

Longer write-up: [Docker daemon socket error](https://www.ryuslog.com/blog/docker-cannot-connect-to-docker-daemon-unix-socket-fix).
Reference: [protect the Docker daemon socket](https://docs.docker.com/engine/security/protect-access/),
[Docker contexts](https://docs.docker.com/engine/manage-resources/contexts/).
