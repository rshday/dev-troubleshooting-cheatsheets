# Kubernetes 1.35: cgroup v1 removal and the node that never joins

A node whose host boots on cgroup v1 does not register with a 1.35 kubelet: the process exits at
startup. Nothing in the cluster explains it, because the failure is on the host. The same release
also removed the `--pod-infra-container-image` kubelet flag, so bootstrap scripts that still pass it
exit with an unknown flag.

## Symptom → first check

| Symptom | Likely cause | First command |
|---|---|---|
| Node never becomes `Ready`, kubelet `Active: failed` | kubelet refused to start on cgroup v1 | `journalctl -u kubelet -n 50` |
| `cgroup v1 is no longer supported; failing to start` | Host hierarchy is v1 | `stat -fc %T /sys/fs/cgroup` |
| `unknown flag: --pod-infra-container-image` | bootstrap user data, nodeadm `NodeConfig`, Karpenter `EC2NodeClass`, custom AMI | grep the node's user data |
| Node joins then goes `NotReady` | Runtime not ready or sandbox image missing | `crictl info`, `systemctl status containerd` |
| `kubelet_cri_losing_support` metric rises | containerd 1.x (last supported in 1.35) | `containerd --version` per node |
| kube-proxy logs a deprecation warning | `mode: ipvs` | plan a switch to nftables |

`stat -fc %T /sys/fs/cgroup` returns `cgroup2fs` on v2 and `tmpfs` on v1. The cause is usually one
kernel argument in the launch template (`systemd.unified_cgroup_hierarchy=0`) or a base image older
than 2022.

## Fixes

```yaml
# kubelet configuration on the node (not a Kubernetes object)
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
failCgroupV1: false
```

Treat that as an escape hatch with a date next to it: a later release removes it. The durable fix is
a new node image that boots cgroup v2 (AL2023 and Bottlerocket do by default), then a rolling
replacement — kernel arguments are read at boot, so an in-place patch needs a reboot anyway.

```bash
# find hosts still on v1
for n in $(kubectl get nodes -o name | cut -d/ -f2); do
  printf "%s " "$n"
  kubectl debug node/$n -it --image=busybox -- \
    sh -c 'stat -fc %T /sys/fs/cgroup' 2>/dev/null | tail -1
done
```

## Upgrade order

1. API servers one at a time (1.35 API server supports kubelets back to 1.32).
2. Controllers.
3. Kubelets node by node, after a drain: `kubectl drain <node> --ignore-daemonsets --delete-emptydir-data --timeout=300s`.
4. kube-proxy no newer than the API server.

Do not roll a node pool whose images still boot cgroup v1: the drain succeeds and the node never
comes back, which turns a routine rotation into an outage.

## Related removals in the same release

- `--pod-infra-container-image` is gone; let the runtime supply the sandbox image.
- containerd 1.x is supported for the last time; the next upgrade requires 2.0+.
- Ingress NGINX is archived after March 2026 — count `nginx.ingress.kubernetes.io/*` annotations per
  Ingress rather than objects, since some have no Gateway API equivalent.

Reference: <https://kubernetes.io/docs/concepts/architecture/cgroups/>
