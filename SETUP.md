# Setup Guide: Bare-Metal / Mini-PC Kubernetes

These patterns are distro-agnostic — they apply to any Kubernetes cluster, cloud or self-hosted. This guide covers the parts specific to running on your own hardware, where a cloud provider isn't handling networking, storage, or node provisioning for you.

## 1. Choose an OS

Two reasonable options for a from-scratch home cluster:

- **[Talos Linux](https://www.talos.dev)** — immutable, API-driven, no SSH, config applied declaratively via `talosctl`. More setup friction up front (no shell to debug from); in exchange, drift between nodes becomes structurally hard to introduce.
- **A minimal Linux distro + kubeadm** — more familiar if you already know standard Linux administration; you're responsible for keeping every node's OS state consistent yourself.

Whichever you pick, plan for an odd number of control-plane nodes (3 is the practical minimum for etcd quorum tolerating one node down) and put resource requests/limits on everything from day one — noisy-neighbor problems on identical mini-PC hardware are immediate and obvious in a way they aren't on elastic cloud instances.

## 2. Networking

- **CNI**: Flannel is the simplest correct choice for a flat home network. Verify each node's CNI interface (e.g. `flannel.1`) has the *correct* node IP in its public-IP annotation after any node re-provision — a stale annotation here causes cross-node pod traffic to silently fail with no obvious error at the application layer.
- **Ingress**: see [`ingress/`](./ingress/) for the Cloudflare Tunnel pattern — no cloud load balancer, no inbound ports opened on your router.

## 3. Storage

Local-path-provisioner is the pragmatic default for single-node-affinity workloads. For anything that needs to survive a node failure, you need either a distributed storage layer (Longhorn, Rook/Ceph) or an explicit backup strategy to off-cluster storage (object storage, e.g. via `restic` or `velero`) — local-path alone means data lives and dies with one specific node.

## 4. Observability

See [`observability/`](./observability/) — in particular, if you're on Talos Linux, `kube-prometheus-stack`'s dashboards for etcd/scheduler/controller-manager/kube-proxy ship empty by default until you apply the control-plane metrics patch documented there. This is easy to miss because nothing errors — the scrape targets are just silently unreachable.

## 5. Deploy your first workload

```bash
cd helm/sample-service
helm install sample-service . -f values-dev.yaml
```

See the main [README](./README.md) for the full pattern list and the reasoning behind each one.
