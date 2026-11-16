# Kubernetes: OOMKilled, exit code 137, and probe timing

## 1. Read the termination reason before changing anything

```bash
kubectl get pod my-pod -o jsonpath='{.status.containerStatuses[*].lastState.terminated}' | jq
kubectl describe pod my-pod | sed -n '/Last State/,/Ready/p'
```

| What you see | What it means | Where to look |
|---|---|---|
| `reason: OOMKilled`, `exitCode: 137` | The **kernel** killed the container's main process for exceeding its memory limit | Container `limits.memory`, real RSS, JVM/Node heap |
| `exitCode: 137`, no `OOMKilled` | SIGKILL from something else — node pressure eviction, kubelet, `kubectl delete` | `kubectl get events`, node `MemoryPressure` |
| `exitCode: 143` | SIGTERM handled or ignored until the grace period ended | Lifecycle hooks, `terminationGracePeriodSeconds` |
| `reason: Error`, `exitCode: 1` | The app exited on its own | App logs, not the limit |
| `reason: Completed`, `exitCode: 0` | Normal exit — a process that shouldn't be in a Deployment | Command/entrypoint |

`OOMKilled` is a **cgroup memory** event: the container crossed `limits.memory` and the kernel's
OOM killer chose a process inside it. It is not the same as a node-level eviction, which shows
up as `Evicted` with `The node was low on resource: memory`.

## 2. Requests, limits, and QoS — the part that decides who dies first

```yaml
resources:
  requests:            # what the scheduler reserves; used for QoS and eviction order
    memory: 512Mi
    cpu: 250m
  limits:              # the ceiling; memory over the limit = OOMKill
    memory: 512Mi
```

| requests vs limits | QoS class | Eviction priority under node pressure |
|---|---|---|
| both unset | BestEffort | first to go |
| set, but `requests < limits` | Burstable | middle |
| `requests == limits` for memory and CPU | Guaranteed | last |

Two rules that avoid most of the pain:

- **CPU limits are throttling, not safety.** CPU over the limit is throttled (`container_cpu_cfs_throttled_seconds_total`), never killed. Many teams set CPU requests and drop CPU limits.
- **Memory limits are a hard kill.** There is no throttle for memory. Set `requests.memory` to what the process needs at steady state and leave real headroom in `limits.memory`.

If you want to understand how the numbers map to `OOMKilled` in a specific workload,
[Resource requests, limits, QoS classes and OOMKilled](https://www.ryuslog.com/blog/kubernetes-resource-requests-limits-qos-classes-and-oomkilled)
walks the same table with worked examples.

## 3. Finding the real memory usage

```bash
# current usage of every container in the namespace
kubectl top pod --containers -n prod

# 30-second-resolution history, per container
kubectl get --raw "/apis/metrics.k8s.io/v1beta1/namespaces/prod/pods" | jq \
  '.items[] | {pod: .metadata.name, mem: [.containers[] | {n: .name, m: .usage.memory}]}'
```

The metric to compare against the limit is **working set**, not `rss` alone — page cache that the
kernel can reclaim is part of the cgroup's charge for a while.

Typical causes of a slow climb to the limit:

| Cause | Signature | Fix |
|---|---|---|
| Unbounded cache / map keyed by user input | heap grows monotonically, never returns | cap it (LRU, TTL), or set a max size |
| Runtime heap larger than the container limit | JVM `-Xmx` or `--max-old-space-size` > ~75% of `limits.memory` | set the heap to 60–75% of the limit; leave room for metaspace, threads, direct buffers |
| Leak in native code / off-heap buffers | RSS grows, heap stable | `pmap`, `jemalloc` stats, upgrade the library |
| Batch job loading everything at once | spike then kill | stream/chunk, or raise the limit for that Job only |
| Many concurrent requests buffering bodies | memory tracks request rate | limit body size, stream uploads |

For the JVM:

```bash
# the container-aware defaults, if you set nothing
java -XX:+PrintFlagsFinal -version | grep -E 'MaxHeapSize|MaxRAMPercentage'
```

Modern JDKs read the cgroup limit and default `MaxRAMPercentage=25`. On a 512Mi limit that is a
128Mi heap — enough to look fine in dev and die in prod, or the reverse.

## 4. Probes: the three you need and the order they run

```yaml
startupProbe:            # runs first; while it fails, liveness is disabled
  httpGet: { path: /healthz, port: 8080 }
  failureThreshold: 30
  periodSeconds: 2       # up to 60s to boot
livenessProbe:           # "restart me" — only for a wedged process
  httpGet: { path: /healthz, port: 8080 }
  periodSeconds: 10
  failureThreshold: 3
readinessProbe:          # "take me out of the Service" — for warmth and dependencies
  httpGet: { path: /ready, port: 8080 }
  periodSeconds: 5
```

| Symptom | Usually |
|---|---|
| `CrashLoopBackOff` right after a slow start, exit 143 | liveness killed it while it was still booting → add/adjust a `startupProbe` |
| Pod `Running` but traffic never arrives | readiness never passes: dependency check, wrong port, probe path returns 404 |
| Restarts exactly every N seconds, healthy in between | liveness probe too aggressive or hitting a saturated thread pool |
| Both probes restarted the app during a GC pause | liveness timeout too tight; use readiness for slow, liveness only for dead |

Rules of thumb:

- **Liveness should not check dependencies.** A database outage should make pods *not ready*, not restart in a loop — restarts cannot fix a database.
- **`initialDelaySeconds` is a blunt instrument.** Prefer `startupProbe`; it stops liveness while booting and does not penalise later slow responses.
- **`timeoutSeconds` must be shorter than `periodSeconds`**, and `failureThreshold × periodSeconds` is your real detection window.
- **Graceful shutdown interacts with readiness.** On SIGTERM, fail readiness first, then drain (`terminationGracePeriodSeconds` > longest in-flight request + preStop sleep).

The full timing table — including what a restart does to `CrashLoopBackOff` backoff and how
`failureThreshold` compounds — is in
[Liveness, readiness and startup probes](https://www.ryuslog.com/blog/kubernetes-liveness-readiness-and-startup-probes).

## 5. Triage order for a pod that will not stay up

```text
1. kubectl describe pod        → Last State / exit code / Events      (what killed it)
2. kubectl logs --previous     → app's own last words (always --previous)
3. kubectl top pod             → memory at the moment of death if it survived long enough
4. kubectl get events --sort-by=.lastTimestamp → evictions, probe failures, image pulls
5. node view                   → dmesg | grep -i oom, node conditions
```

---

Related: [Linux OOM killer and swap tuning](https://www.ryuslog.com/blog/linux-swap-memory-oom-killer-vmstat-and-sysctl-tuning) ·
[Jobs and CronJobs: completions, parallelism, backoff](https://www.ryuslog.com/blog/kubernetes-jobs-and-cronjobs-completions-parallelism-and-backoff) ·
Live YAML/JSON checks in the browser: [YAML ↔ JSON](https://www.ryuslog.com/tools/yaml)
