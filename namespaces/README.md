# Namespace Isolation: RBAC + NetworkPolicy per Team

The pattern: every team gets a namespace, and two boundaries are enforced from the start — network (default-deny, explicit allow) and access (`Role` scoped to that one namespace, never cluster-wide).

Both boundaries matter independently. RBAC without NetworkPolicy means a compromised pod in one team's namespace can still reach every other service on the cluster network, regardless of who's allowed to *deploy* to it. NetworkPolicy without RBAC means the network is locked down but anyone with `kubectl` access can still touch every namespace.

## Apply

```bash
kubectl apply -f team-namespace/00-namespace.yaml
kubectl apply -f team-namespace/01-networkpolicy-default-deny.yaml
kubectl apply -f team-namespace/02-rbac.yaml
```

## Why default-deny first, not last

Retrofitting `default-deny-all` onto a namespace that's already running workloads breaks things immediately and untraceably — every pod-to-pod call that was implicitly allowed now fails, and the failure looks like a connectivity bug, not a policy change, unless you already know to check NetworkPolicies. Applying it before any workload exists means every subsequent allow-rule is a deliberate decision, not damage control.

## Extending this per additional team

Duplicate `team-namespace/` per team, changing the `team` label and all `namespace:` fields consistently. If two teams need to talk to each other, that's an explicit `NetworkPolicy` allow-rule referencing the other namespace's label — never a blanket loosening of the default-deny baseline.
