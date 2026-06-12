# Phase 2 — Kubernetes Beyond Basics

## Scope
Cluster + namespaces, Redis StatefulSet, HPA with a custom metric, PDBs,
NetworkPolicy, VPA in recommendation mode, and a ResourceQuota.

## Implementation

### Cluster + namespaces (Tasks 6, 7)
- `major-project/deploy/kind/kind-cluster.yml` — 1 control-plane + 2 workers.
- Namespaces `app` (`platform/namespaces/app.yaml`), `monitoring` and `gitops`
  (`bootstrap/namespaces/`).

### Redis StatefulSet (Task 8)
`base/redis/statefulset.yaml` — `StatefulSet` (not Deployment), 1 replica,
`volumeClaimTemplates` → PVC `redis-data` (1Gi, `standard`). Headless +
ClusterIP Services in `base/redis/`.

### HPA with custom metric (Task 9)
`base/api-service/hpa.yaml` — min 2 / max 6; primary `Resource cpu`
`averageUtilization: 60`; secondary `Pods` metric `http_requests_per_second`
`averageValue: 10`. The custom metric is served by **prometheus-adapter**
(`monitoring/prometheus-adapter-values.yaml`) over `custom.metrics.k8s.io`,
derived from `http_requests_total`.

### PodDisruptionBudgets (Task 10)
`base/{api,worker,chaos}-service/pdb.yaml` — `minAvailable: 1` each. With 2
replicas this yields `ALLOWED DISRUPTIONS = 1`.

### NetworkPolicy (Task 11)
`base/netpol-deny-all.yaml` (default-deny-ingress, all pods),
`base/netpol-allow-redis.yaml` (ingress to `app=redis:6379` from `api-service`
and `worker-service`), `base/netpol-allow-prometheus.yaml` (ingress to `:8000`
from the `monitoring` namespace). **Enforcement is real** on this kindnet build
(verified: an unlabelled/disallowed pod times out to Redis; allowed pods connect).

### VPA recommendation mode (Task 12)
`base/chaos-service/vpa.yaml` — `updatePolicy.updateMode: "Off"` (recommendation
only). The VPA recommender controller + CRD are installed from
`major-project/deploy/k8s/vpa/` (not Argo-managed — a minor GitOps deviation).

### ResourceQuota (Task 2.5)
`base/resourcequota.yaml` — `requests.cpu: "2"`, `requests.memory: 1Gi`.
Sized tight on purpose: steady state is 896Mi (7 pods × 128Mi), so a scale-out or
rolling-update surge that needs an 8th pod hits the limit and the quota rejects
it with an `exceeded quota` event — exactly the Phase 2.5 behaviour.

## Commands / validation
```bash
kubectl get nodes
kubectl get statefulset,pvc -n app
kubectl describe hpa api-service -n app
kubectl get pdb -n app
kubectl get networkpolicy -n app
kubectl describe resourcequota app-quota -n app
```
NetworkPolicy proof and the rest of the read-only checks are in
`docs/validation-guide.md`.

## Evidence
- Available: all manifests; live HPA showing a real custom-metric value; PDBs at
  `ALLOWED DISRUPTIONS=1`; **real `exceeded quota` FailedCreate events** captured
  during a staging/prod rollout.
- Captured (held by author): PDB node-drain demo, NetworkPolicy rejected-probe
  screenshot, HPA scaling, and the VPA recommendation output — collect these into
  `docs/evidence/` for the submission bundle.

## Status
Done, with the noted demo captures outstanding and the VPA controller installed
imperatively.
