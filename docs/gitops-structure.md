# GitOps repository structure

## App of Apps

A single root `Application` (`apps/root-app.yaml`) watches the `apps/` directory
of this repo and manages the child Applications declared there as plain
manifests (Argo CD "directory" source — no Kustomize needed for this path):

| Application | Manages | Source path | Destination namespace |
|---|---|---|---|
| `root-app` | the two child Applications below | `apps/` | `gitops` |
| `monitoring-app` | Prometheus Operator, Grafana, Prometheus Adapter | Helm charts (`kube-prometheus-stack`, `prometheus-adapter`) + values from `monitoring/` | `monitoring` |
| `app-app` | the three task-queue services + Redis | `overlays/dev` (→ `base/`) | `app` |

This means the root Application is itself the *only* thing bootstrapped
outside of Argo CD's reach (it was created once via `argocd app create` /
`kubectl apply`, pointed at this repo) — everything below it, including its
own child Application definitions, is reconciled from git from that point on.

## Why the sync-wave ordering matters

Argo CD syncs lower `argocd.argoproj.io/sync-wave` values first, and waits for
each wave to reach a healthy state before starting the next. We use it at two
levels:

**1. Between the child Applications** (`monitoring-app` wave `-1`, `app-app`
wave `0`): the `app` namespace contains `ServiceMonitor` and
`VerticalPodAutoscaler` custom resources. Their CRDs are *not* defined in this
repo — they're installed by the Prometheus Operator and VPA charts that
`monitoring-app` brings in. If `app-app` synced first (or concurrently), the
API server would reject those custom resources with
`no matches for kind "ServiceMonitor"` because the CRD wouldn't exist yet.
Forcing `monitoring-app` to finish first guarantees the CRDs are registered
before `app-app` tries to create instances of them — this is the practical,
real-world version of "assign wave `-1` to CRDs:" we don't ship raw CRD YAML
in this repo, but we do control the order in which the *Application that
installs them* runs relative to the *Application that depends on them*.

**2. Inside `app-app`'s own resource set** (namespace wave `0`, workloads wave
`1`): the `Namespace/app` manifest (`platform/namespaces/app.yaml`) must exist
before the API server will accept any namespaced resource — Deployments,
the Redis StatefulSet, Services, NetworkPolicies, the ResourceQuota, and so
on — that declares `namespace: app`. Without an explicit wave, Argo CD applies
everything in one pass and the ordering is left to chance (it usually works,
because the namespace happens to apply quickly, but it's not guaranteed —
under load or API-server backpressure the workloads can race the namespace
and fail with `namespaces "app" not found`). Pinning the namespace to wave `0`
and every workload to wave `1` makes that guarantee explicit and reproducible.

## Adopting the monitoring stack into Argo CD

`monitoring-app` is committed but **its `syncPolicy` is intentionally not
`automated` yet**. The `kube-prometheus-stack` and `prometheus-adapter`
releases currently running in the `monitoring` namespace were installed by
hand with `helm install` (visible via `helm list -A`) — they are not yet
managed by Argo CD, which is itself a gap against the "GitOps only" rule.

Pointing an automated Argo CD Application at the same namespace immediately
is risky: Argo CD renders manifests with `helm template` (it does not use
Helm's release-secret bookkeeping), so the two mechanisms would both claim
ownership of the same Prometheus/Grafana/Alertmanager StatefulSets and
Deployments, and the resulting reconciliation could restart the whole
monitoring stack at an unpredictable moment — exactly when you need Grafana
and Prometheus up for evidence captures in Phases 5 and 6.

**Cutover plan (do this deliberately, in a maintenance window, not mid-demo):**

```bash
# 1. Confirm the chart versions/values in monitoring-app.yaml match what's live
helm -n monitoring get values kube-prometheus-stack
helm -n monitoring get values prometheus-adapter

# 2. Remove the manually-installed releases (Argo CD will recreate everything
#    from the same chart versions + the values already committed in monitoring/)
helm -n monitoring uninstall kube-prometheus-stack
helm -n monitoring uninstall prometheus-adapter

# 3. Flip monitoring-app.yaml's syncPolicy to:
#      automated:
#        prune: true
#        selfHeal: true
#    commit + push, then either wait for root-app's auto-sync or force it:
argocd app sync monitoring-app
```

Expect a brief window (a few minutes) where Prometheus/Grafana/Alertmanager
are unavailable while the new pods come up. Plan this around your evidence
capture schedule, not in the middle of it.
