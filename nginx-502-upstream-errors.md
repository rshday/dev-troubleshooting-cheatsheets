# nginx 502 Bad Gateway: read the upstream error line

A 502 says one thing at the top level: nginx accepted the client connection and could not get a
valid response from the upstream. Everything useful is in the error log, which names the phase
that failed — connect, send, or reading the response header.

```
[error] 31#31: *7 connect() failed (111: Connection refused) while connecting to upstream
[error] 31#31: *9 upstream prematurely closed connection while reading response header
```

## Log line to cause

| Error log text | Cause | Fix |
| --- | --- | --- |
| `connect() failed (111: Connection refused)` | nothing listening at that host and port | correct the host/port, or start the upstream |
| `connect() failed (110: Connection timed out)` | packets dropped: wrong network, firewall, security group | check routes and service exposure |
| `upstream prematurely closed connection` | upstream died or timed out mid-response | read upstream logs, raise its timeout |
| `no live upstreams while connecting` | every backend fails its passive health checks | fix one backend, tune `max_fails` |
| `upstream sent too big header` | response header larger than the buffer | raise `proxy_buffer_size` |
| `recv() failed (104: Connection reset by peer)` | kernel reset, often a half-closed keepalive socket | align keepalive settings on both sides |
| `SSL_do_handshake() failed` | TLS mismatch to the upstream | fix the scheme and SNI |
| `upstream timed out ... while reading response header` (client sees 504) | app slower than `proxy_read_timeout` | align app timeout, `proxy_read_timeout`, and the LB in front |

## The mistake that produces most of them: localhost across containers

Inside a container, `127.0.0.1` is that container. A `proxy_pass http://127.0.0.1:3000` in an nginx
container therefore targets nginx itself and fails with errno 111.

```nginx
# wrong inside a container
location / { proxy_pass http://127.0.0.1:3000; }

# right: the Compose service name on the shared network
location / {
    proxy_pass http://app:3000;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
}
```

Test from the namespace that matters — `curl localhost:3000` on the host proves nothing:

```bash
docker compose exec proxy getent hosts app
docker compose exec proxy wget -qO- http://app:3000/
docker compose config | grep -A5 'networks:'
```

## Start-up order is not readiness

`depends_on` waits for the container to start, not for the process to listen. nginx caches the
failed connection and keeps answering 502 until reloaded.

```yaml
services:
  app:
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://localhost:3000/health"]
      interval: 5s
      retries: 10
  proxy:
    depends_on:
      app:
        condition: service_healthy
```

When the upstream IP changes (Kubernetes, ECS, `docker compose up --scale`), a static `proxy_pass`
keeps the first resolved address forever. Use a resolver to re-resolve per request:

```nginx
resolver 127.0.0.11 valid=10s ipv6=off;
set $upstream http://app:3000;
proxy_pass $upstream;
```

## The 502 that only hits logged-in users

A large `Set-Cookie` or a JWT in a response header can exceed the default 4 KB buffer:

```nginx
proxy_buffer_size 16k;
proxy_buffers 4 32k;
proxy_busy_buffers_size 64k;
large_client_header_buffers 4 16k;
```

The same symptom with a **400** instead of 502 is the *client* header being too large, which
`large_client_header_buffers` covers.

## PHP-FPM specifics

With `fastcgi_pass unix:/run/php/php8.3-fpm.sock`, a 502 usually means the pool's socket path
differs from nginx's, or the pool exhausted its children:

```bash
ss -xl | grep php
grep -E 'listen|pm.max_children' /etc/php/8.3/fpm/pool.d/www.conf
tail -50 /var/log/php8.3-fpm.log
```

Raise `pm.max_children` when the log says `server reached pm.max_children`, and check
`request_terminate_timeout` against nginx's `fastcgi_read_timeout`.

## Things that are not the cause

- **nginx being broken.** It is reporting an upstream failure; the upstream is the subject.
- **The nginx version.** A changed upstream port or a new health endpoint is far more likely than
  a regression after an upgrade.
- **A reload without a resolver after a DNS change.** Static upstream names are resolved at
  start-up only.
- **Editing `/etc/nginx/nginx.conf` in the official image.** `/etc/nginx/conf.d/default.conf`
  already has a `server` block on port 80 that shadows yours.

A 502 that disappears when you test from the host is a network-scope problem every time.

Longer write-up: [nginx 502 Bad Gateway](https://www.ryuslog.com/blog/nginx-502-bad-gateway-upstream-connection-refused).
Reference: [ngx_http_proxy_module](https://nginx.org/en/docs/http/ngx_http_proxy_module.html),
[ngx_http_upstream_module](https://nginx.org/en/docs/http/ngx_http_upstream_module.html).
