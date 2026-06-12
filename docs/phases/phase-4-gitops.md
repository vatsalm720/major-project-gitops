# Phase 4 — GitOps with Argo CD

## Scope
App-of-Apps, sync waves, drift detection/auto-heal, ApplicationSet across
overlays, and SealedSecrets.

## Implementation

### Repository layout (Task 2.8)
`apps/`, `base/`, `overlays/{dev,staging,prod}/`, `monitoring/` — matches the
required structure. A static `app-app.yaml` is replaced by an `ApplicationSet`
that generates the per-environment Applications (Task 22), which is a superset of
the requirement.

### App-of-Apps (Task 19)
`apps/root-app.yaml` is the root Application; it watches `apps/` and manages four
children: `monitoring-app`, `sealed-secrets-app`, `applicationset`
(→ dev/staging/prod), and `webhook-listener-app`. `root-app` omits an explicit
`directory.recurse` block so the live object matches git (otherwise Argo CD
normalises `recurse:false` to null and the root shows perpetually OutOfSync).

### Sync waves (Task 20)
- wave **-1**: `monitoring-app` (Prometheus Operator CRDs) and
  `sealed-secrets-app` (SealedSecret CRD + controller).
- wave **0**: namespaces, `applicationset`, `webhook-listener-app`, SealedSecrets.
- wave **1**: service Deployments / Services / Redis / NetworkPolicies / quota.
- wave **2**: api-service HPA (needs its target Deployment first).

Ordering matters because `ServiceMonitor`/`PrometheusRule`/`SealedSecret` are
CRDs — applying instances before their CRDs exist fails the sync.

### Drift detection + auto-heal (Task 21)
Every Application sets `syncPolicy.automated: { prune: true, selfHeal: true }`.
A live edit (e.g. `kubectl scale`) shows OutOfSync and is reverted automatically.
This behaviour was also observed during incident recovery (self-heal reverting a
manual change).

### ApplicationSet (Task 22)
`apps/applicationset.yaml` — a single list generator + template producing
`dev-app` (ns `app`), `staging-app` (ns `app-staging`), `prod-app` (ns
`app-prod`), each pointing at `overlays/<env>`. Staging/prod overlays patch the
base `ServiceMonitor` `namespaceSelector` to their own namespace and pin images.

### SealedSecrets (Task 2.10)
`apps/sealed-secrets-app.yaml` installs the controller (Helm). Only the encrypted
`overlays/dev/app-credentials-sealed.yaml` is committed — no plaintext secret in
git history. Fresh-cluster decryption requires restoring the controller key or
re-sealing (see `docs/deployment-guide.md`).

## Commands / validation
```bash
kubectl get applications -n gitops
kubectl kustomize overlays/dev && kubectl kustomize overlays/staging && kubectl kustomize overlays/prod
# drift demo (allowed live edit)
kubectl -n app scale deploy/api-service --replicas=5
kubectl get application dev-app -n gitops -w
```

## Evidence
- Available: all `apps/` manifests; `kubectl get applications` showing all
  Synced+Healthy; clean local renders of all three overlays.
- Captured (held by author): OutOfSync + auto-heal drift screenshots,
  ApplicationSet and sync-wave screenshots — collect into `docs/evidence/`.
- Outstanding: **fresh-cluster SealedSecret decryption** validation (controller
  key restore vs re-seal).

## Status
Done; the fresh-cluster secret test is the only open item.
