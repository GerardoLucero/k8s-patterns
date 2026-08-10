# Control-Plane Metrics on Talos Linux

**The problem:** on Talos Linux, `kube-scheduler`, `kube-controller-manager`, and `etcd` bind their metrics endpoints to `127.0.0.1` on the node by default — unreachable from `kube-prometheus-stack`, which runs as a pod in another namespace. `kube-proxy` has the same issue: no `--metrics-bind-address` set, so it defaults to loopback too.

The result isn't a crash — it's silent. Everything looks healthy, and Prometheus quietly can't scrape four of your most important components. In one real cluster this ran for **44 days** before anyone noticed, because the alerts that depend on that data (`etcdInsufficientMembers`, `KubeSchedulerInstanceUnreachable`, `KubeProxyInstanceUnreachable`) were firing constantly as false positives from the connection failures themselves — noise masking the actual gap.

**Symptom check:**

```bash
# From any machine that can reach the node IP directly:
curl -sk https://<control-plane-ip>:10259/metrics   # kube-scheduler
curl -sk https://<control-plane-ip>:10257/metrics   # kube-controller-manager
curl -s  http://<control-plane-ip>:2381/metrics     # etcd
curl -s  http://<any-node-ip>:10249/metrics          # kube-proxy
```

If these hang or return `connection refused`, this is your issue.

## The fix

### 1. `kube-scheduler` + `kube-controller-manager` + `etcd`

These are Talos machine config, not standard Kubernetes objects. Apply [`talos-controlplane-metrics-patch.yaml`](./talos-controlplane-metrics-patch.yaml) to **each control-plane node, one at a time** — not all at once, so you keep etcd quorum through the change:

```bash
talosctl patch machineconfig --patch @talos-controlplane-metrics-patch.yaml \
  -e <control-plane-ip> -n <control-plane-ip> \
  --talosconfig=<path-to-talosconfig>
```

`scheduler` and `controller-manager` pick up the change immediately — Talos recreates the static pod, no reboot. **`etcd` does not** — the process doesn't support a hot restart via the Talos API (`service "etcd" doesn't support restart operation`), so the node needs a full reboot for `listen-metrics-urls` to take effect:

```bash
talosctl -e <control-plane-ip> -n <control-plane-ip> reboot
```

Before rebooting each node, confirm etcd has 3 healthy members (or however many you run) so quorum survives one node going down:

```bash
talosctl -e <control-plane-ip> -n <cp1>,<cp2>,<cp3> etcd status
```

Repeat per control-plane node. Verify after each reboot that quorum is restored before moving to the next one.

### 2. `kube-proxy`

Not Talos machine config — it's a normal `DaemonSet`, patched directly:

```bash
kubectl patch daemonset kube-proxy -n kube-system --type='json' -p='[
  {"op":"add","path":"/spec/template/spec/containers/0/command/-",
   "value":"--metrics-bind-address=0.0.0.0:10249"}
]'
kubectl rollout status daemonset kube-proxy -n kube-system
```

## Verification

```bash
# Re-run the symptom-check curls above — scheduler/controller-manager
# should now return 403 (reachable, just needs the bearer token
# Prometheus sends), and etcd/kube-proxy should return 200.
```

Then check Prometheus directly:

```promql
etcd_server_has_leader
scheduler_schedule_attempts_total
kubeproxy_sync_proxy_rules_duration_seconds_count
```

All three should return series once the ServiceMonitors (already shipped with `kube-prometheus-stack`) start scraping successfully — no ServiceMonitor changes needed, the Services already point at the right ports, they just couldn't reach anything before.

## Why this matters beyond "one more metric"

This isn't cosmetic. `etcdInsufficientMembers` is a **critical**-severity alert — the kind that's supposed to mean "your cluster's source of truth is failing." When the underlying scrape target has been unreachable for weeks, that alert fires constantly for the wrong reason, and the team (or the on-call you) stops trusting it. The day etcd actually degrades, the signal looks identical to the noise that's been there the whole time. Fixing the scrape target isn't about having a prettier dashboard — it's about making a **critical** alert mean something again.

## Security note

`etcd`'s metrics endpoint is unauthenticated once exposed on `0.0.0.0:2381`. This is fine on a private, firewalled network (the standard home-lab / internal-VPC case) — it is not something you'd expose on a public interface without putting a NetworkPolicy or firewall rule in front of it.
