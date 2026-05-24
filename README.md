# Kubernetes Patterns

Production-ready Kubernetes patterns for platform engineers. Covers multi-environment Helm charts, RBAC, ingress, resource management, and observability — patterns I use daily operating a self-hosted K8s cluster running AI workloads.

[![Ko-fi](https://img.shields.io/badge/Ko--fi-FF5E5B?style=flat&logo=ko-fi&logoColor=white)](https://ko-fi.com/gerardolucero)
[![Buy Me a Coffee](https://img.shields.io/badge/Buy%20Me%20a%20Coffee-FFDD00?style=flat&logo=buy-me-a-coffee&logoColor=black)](https://buymeacoffee.com/lucerorios0)
[![GitHub Stars](https://img.shields.io/github/stars/GerardoLucero/k8s-patterns?style=social)](https://github.com/GerardoLucero/k8s-patterns)

## What's Covered

| Pattern | Description | Status |
|---|---|---|
| [Multi-env Helm Charts](./helm/) | Dev / Staging / Prod with environment-specific values | ✅ |
| [Namespace Isolation](./namespaces/) | RBAC + NetworkPolicy per team | ✅ |
| [Ingress + TLS](./ingress/) | Traefik + cert-manager + Let's Encrypt | ✅ |
| [Resource Management](./resources/) | Requests, limits, VPA, PodDisruptionBudgets | 🔄 |
| [Persistent Volumes](./storage/) | PVC patterns for stateful workloads | 🔄 |
| [Observability Stack](./observability/) | kube-prometheus-stack + Grafana dashboards | 🔄 |
| [Agent Workloads](./agents/) | Running AI agents as K8s Jobs + KEDA autoscaling | 🔄 |

## Why These Patterns

I operate a 3-node k3s cluster at home (AMD Ryzen 5, 16GB RAM per node) hosting AI agent workloads. These are the patterns that work in a real environment — not just on managed cloud K8s.

Real constraints I solve for:
- No cloud load balancer → Traefik Ingress + Cloudflare Tunnel
- No NFS server → LocalPath provisioner with backup strategy
- Multiple AI workloads competing for GPU/CPU → proper resource limits and KEDA autoscaling
- Zero-downtime deploys on limited hardware → PodDisruptionBudgets + rolling update strategy

## Stack

```
Cluster: k3s (lightweight K8s)
Ingress: Traefik (built-in with k3s)
TLS: cert-manager + Let's Encrypt
Networking: Flannel CNI + Cloudflare Tunnel
Storage: Local-path-provisioner
Monitoring: kube-prometheus-stack (Prometheus + Grafana + Alertmanager)
Autoscaling: KEDA for event-driven workloads
CI/CD: GitHub Actions → Docker Hub → kubectl apply
```

## Quick Start

```bash
# Prerequisites: kubectl configured, Helm installed

# Add required repos
helm repo add traefik https://helm.traefik.io/traefik
helm repo add jetstack https://charts.jetstack.io
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install cert-manager
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --set installCRDs=true

# Deploy a sample workload with full stack
cd helm/sample-service
helm install sample-service . -f values-dev.yaml
```

## Pattern 1: Multi-Environment Helm Charts

The core pattern: one chart, multiple `values-*.yaml` files. No Kustomize complexity, no duplication.

```
helm/sample-service/
├── Chart.yaml
├── values.yaml          # defaults + documentation
├── values-dev.yaml      # dev overrides (low replicas, relaxed limits)
├── values-staging.yaml  # staging (mirrors prod sizing)
├── values-prod.yaml     # prod (full replicas, strict limits, PDB)
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    ├── hpa.yaml
    └── pdb.yaml
```

Key decisions:
- **No templating of template logic** — keep `values.yaml` as the source of truth for what changes between environments
- **PodDisruptionBudget in prod only** — `pdb.yaml` has `{{ if eq .Values.env "prod" }}` guard
- **Resource requests = limits in prod** — prevents noisy-neighbor on limited hardware

## Pattern 2: Agent Workloads with KEDA

Running AI agents as Kubernetes Jobs that scale to zero when idle:

```yaml
# agents/nexus-agent/scaledjob.yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledJob
metadata:
  name: nexus-daily-focus
spec:
  jobTargetRef:
    template:
      spec:
        containers:
        - name: agent
          image: ghcr.io/gerardolucero/nexus-agent:latest
          env:
          - name: TASK
            value: "daily-focus"
  triggers:
  - type: cron
    metadata:
      timezone: America/Mexico_City
      start: "0 8 * * *"
      end: "30 8 * * *"
      desiredReplicas: "1"
```

Why Jobs instead of Deployments: AI agents are ephemeral — they process a task and exit. Jobs fit this model better than long-running Deployments, and KEDA's `ScaledJob` handles the lifecycle automatically.

## Pattern 3: Ingress + TLS Without Cloud Load Balancer

On bare-metal / home lab, you don't have a cloud load balancer. Cloudflare Tunnel solves this without opening ports:

```
Internet → Cloudflare → cloudflared (Pod in cluster) → Traefik → Services
```

```yaml
# ingress/cloudflare-tunnel/deployment.yaml
spec:
  containers:
  - name: cloudflared
    image: cloudflare/cloudflared:latest
    args:
    - tunnel
    - --config
    - /etc/cloudflared/config.yaml
    - run
```

No `hostPort`, no `LoadBalancer` type, no router port-forwarding required.

## Observability

Everything uses kube-prometheus-stack. Custom dashboards track:
- Node CPU/RAM saturation (alert at 80%)
- Per-workload resource usage
- AI agent job duration and success rate
- API response time per service

```bash
# Access Grafana locally
kubectl port-forward svc/kube-prometheus-stack-grafana 3000:80 -n monitoring
# Default: admin / prom-operator
```

## Architecture Decisions

**Why k3s over kubeadm?**  
Single binary, automatic TLS, built-in Traefik and CoreDNS. On home lab hardware with no dedicated ops time, simplicity wins over control.

**Why not managed K8s (EKS/GKE)?**  
Cost and learning. Running your own cluster forces you to understand networking, storage, and failure modes that managed K8s abstracts away. If you're interviewing for Platform Engineer roles, this is the difference between knowing K8s and understanding K8s.

**Why KEDA over native HPA for agents?**  
HPA scales Deployments based on CPU/memory. KEDA scales Jobs based on external triggers (cron, queues, events). AI agent workloads are event-driven, not CPU-bound — KEDA is the right tool.

## Setup Guide

See [SETUP.md](./SETUP.md) for step-by-step cluster setup on bare-metal / mini PCs.

## License

MIT
