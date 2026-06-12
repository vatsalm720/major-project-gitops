# major-project-gitops — GitOps Source of Truth

Argo CD watches this repository and reconciles the entire platform onto a local
`kind` cluster. Nothing is deployed with `kubectl apply -f` except the one-time
bootstrap seed (the Argo CD install and the root Application). All application,
monitoring, and policy state is declared here.

Paired application/source repo: `major-project` (builds the images, runs CI).

## Layout

```
apps/                       # App-of-Apps: root + child Applications
  root-app.yaml             #   root Application (watches apps/)
  monitoring-app.yaml       #   kube-prometheus-stack + prometheus-adapter + rules + dashboard (wave -1)
  sealed-secrets-app.yaml   #   Sealed Secrets controller (wave -1)
  applicationset.yaml       #   generates dev-app / staging-app / prod-app (wave 0)
  webhook-listener-app.yaml #   Grafana alert webhook receiver (wave 0)
base/                       # Kustomize base for the 3 services + Redis + policies
  api-service/ worker-service/ chaos-service/ redis/
  configmap.yaml netpol-*.yaml resourcequota.yaml
overlays/                   # per-environment Kustomize overlays
  dev/ staging/ prod/       #   image pins, namespace, SealedSecret (dev)
monitoring/                 # Helm values + rules + dashboard + webhook listener
  kube-prometheus-stack-values.yaml  prometheus-adapter-values.yaml  argocd-values.yaml
  prometheus-rules/  grafana/  webhook-listener/
platform/namespaces/        # app namespace (sync-wave 0)
bootstrap/namespaces/       # gitops + monitoring namespaces (applied at bootstrap)
docs/                       # this documentation set
```

## App-of-Apps and sync waves

`root-app` (Argo CD Application) watches `apps/` and adopts the child
Applications. Sync waves enforce ordering so dependencies exist before dependents:

| Wave | Resources | Why |
|---|---|---|
| **-1** | `monitoring-app` (Prometheus Operator CRDs), `sealed-secrets-app` (SealedSecret CRD + controller) | CRDs must exist before any `ServiceMonitor`/`PrometheusRule`/`SealedSecret` is applied |
| **0** | namespaces, `applicationset`, `webhook-listener-app`, SealedSecret resources | namespaces/secrets land before workloads |
| **1** | service Deployments, Services, Redis StatefulSet, NetworkPolicies, quota | workloads |
| **2** | api-service HPA | the HPA needs its target Deployment to exist first |

The `ApplicationSet` generates one Argo CD Application per environment
(`dev`/`staging`/`prod`) from a single template, each pointing at
`overlays/<env>` and a distinct namespace (`app`/`app-staging`/`app-prod`).

## From-scratch bootstrap (fresh laptop)

Prerequisites: Docker, `kind`, `kubectl`, `helm`, and (for image builds) the
`major-project` repo checked out alongside this one.

> **GitOps-only note.** The only imperative steps are bootstrapping the cluster,
> the local registry, the two seed namespaces, the Argo CD install, and applying
> `apps/root-app.yaml`. After that, Argo CD owns all reconciliation. You cannot
> GitOps the GitOps controller itself — this is the standard App-of-Apps seed.

```bash
# 1. Cluster (1 control-plane + 2 workers)
kind create cluster --name major-project-cluster \
  --config ../major-project/deploy/kind/kind-cluster.yml
kubectl get nodes            # expect 3 Ready

# 2. Local image registry (CI pushes here; nodes use kind-loaded images)
docker run -d --restart=always -p 5000:5000 --name registry registry:2

# 3. Build + load the service images into kind (imagePullPolicy: Never)
for s in api worker chaos; do
  docker build \
    --build-arg VERSION=1.0.0 \
    --build-arg GIT_SHA=bb225dc \
    --build-arg BUILD_DATE=$(date -u +%Y-%m-%dT%H:%M:%SZ) \
    -t localhost:5000/major-project-$s-service:bb225dc \
    ../major-project/services/$s-service/
  kind load docker-image localhost:5000/major-project-$s-service:bb225dc --name major-project-cluster
done

# 4. Seed namespaces (bootstrap exception)
kubectl apply -f bootstrap/namespaces/        # gitops, monitoring
kubectl apply -f platform/namespaces/         # app  (also reconciled by Argo at wave 0)

# 5. Install Argo CD (chart argo/argo-cd 9.5.19 -> Argo CD v3.4.3)
helm repo add argo https://argoproj.github.io/argo-helm && helm repo update
helm install argocd argo/argo-cd -n gitops --version 9.5.19 \
  -f monitoring/argocd-values.yaml

# 6. Seed the App-of-Apps (the one allowed root apply)
kubectl apply -f apps/root-app.yaml

# 7. Watch everything converge
kubectl get applications -n gitops -w
```

After convergence, all Applications report `Synced` + `Healthy`.

> **SealedSecrets caveat (Task 2.10).** The committed `SealedSecret` is encrypted
> against the *original* cluster's controller key. A brand-new cluster gets a new
> controller key and **cannot decrypt it**. To make a genuine fresh-cluster sync
> work you must either (a) back up the original sealing key Secret
> (`kubectl get secret -n kube-system -l sealedsecrets.bitnami.com/sealed-secrets-key -o yaml`)
> and restore it before the controller starts, or (b) re-seal the secret against
> the new controller with `kubeseal`. See `docs/deployment-guide.md`.

## Access

```bash
# Argo CD UI
kubectl port-forward -n gitops svc/argocd-server 8080:443
kubectl -n gitops get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d; echo

# Grafana (admin / admin)
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80

# Prometheus
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090
```

## Documentation index

| Document | Contents |
|---|---|
| `docs/requirement-matrix.md` | Requirement → implementation → evidence audit |
| `docs/architecture-overview.md` | System architecture, components, data/metric flow |
| `docs/deployment-guide.md` | Detailed bootstrap, image flow, secrets, teardown |
| `docs/validation-guide.md` | How to verify each phase (read-only commands) |
| `docs/evidence-checklist.md` | Submission evidence: captured vs outstanding |
| `docs/demo-walkthrough.md` | 15-minute demo script |
| `docs/phases/phase-1..6-*.md` | Per-phase implementation notes |
| `docs/runbooks/experiment-{1,2,3}-*.md` | Chaos experiment runbooks |
| `docs/gitops-structure.md` | Original GitOps structure design notes |
