# Deployment Guide

End-to-end bootstrap of the platform on a fresh machine, plus image flow,
secrets handling, and teardown. All cluster state after bootstrap is owned by
Argo CD.

## Prerequisites

| Tool | Used for |
|---|---|
| Docker | container runtime, local registry |
| kind | local Kubernetes cluster |
| kubectl | read-only inspection, bootstrap seed, drift demo |
| helm | Argo CD install (and the charts Argo CD pulls) |
| kubeseal (optional) | re-sealing secrets on a new cluster |

Check out both repos side by side:

```
projects/
  major-project/            # images + CI
  major-project-gitops/     # this repo
```

## 1. Cluster

```bash
kind create cluster --name major-project-cluster \
  --config ../major-project/deploy/kind/kind-cluster.yml
kubectl get nodes        # 3 Ready: 1 control-plane + 2 workers
```

## 2. Local registry

The CI `release-build` workflow pushes images to `localhost:5000`. Run a registry
so that flow works; cluster pods themselves use side-loaded images
(`imagePullPolicy: Never`).

```bash
docker run -d --restart=always -p 5000:5000 --name registry registry:2
```

## 3. Images

Pods reference `localhost:5000/major-project-<svc>:bb225dc` with
`imagePullPolicy: Never`, so each node's containerd must already hold the image.
Build and side-load:

```bash
for s in api worker chaos; do
  docker build \
    --build-arg VERSION=1.0.0 \
    --build-arg GIT_SHA=bb225dc \
    --build-arg BUILD_DATE=$(date -u +%Y-%m-%dT%H:%M:%SZ) \
    -t localhost:5000/major-project-$s-service:bb225dc \
    ../major-project/services/$s-service/
  kind load docker-image localhost:5000/major-project-$s-service:bb225dc \
    --name major-project-cluster
done
```

When CI later pushes a new SHA, the `release-build` workflow performs the
`kind load` and the bot commit; Argo CD then rolls the Deployment.

## 4. Seed namespaces

```bash
kubectl apply -f bootstrap/namespaces/    # gitops, monitoring
kubectl apply -f platform/namespaces/     # app (also wave-0 reconciled by Argo)
```

## 5. Install Argo CD

```bash
helm repo add argo https://argoproj.github.io/argo-helm && helm repo update
helm install argocd argo/argo-cd -n gitops --version 9.5.19 \
  -f monitoring/argocd-values.yaml
kubectl -n gitops rollout status deploy/argocd-server
```

`argocd-values.yaml` disables TLS on the server (port-forward access), Dex, and
the notifications controller — minimal footprint for a local cluster.

## 6. Seed the App-of-Apps

```bash
kubectl apply -f apps/root-app.yaml
kubectl get applications -n gitops -w
```

`root-app` adopts `monitoring-app`, `sealed-secrets-app`, the `ApplicationSet`
(→ `dev-app`/`staging-app`/`prod-app`), and `webhook-listener-app`. Wait until
all report `Synced` + `Healthy`. First convergence takes a few minutes
(Prometheus Operator CRDs in wave -1, then workloads).

## Secrets (SealedSecrets)

Only the encrypted `overlays/dev/app-credentials-sealed.yaml` is committed; no
plaintext secret exists anywhere in either repo.

The controller is installed by `sealed-secrets-app`. **A fresh cluster generates
a new controller key and cannot decrypt the committed SealedSecret.** Choose one:

- **Restore the original key (true fresh-cluster sync):** on the original
  cluster, back up the sealing key, then on the new cluster apply it before the
  controller first reconciles:
  ```bash
  # original cluster
  kubectl get secret -n kube-system \
    -l sealedsecrets.bitnami.com/sealed-secrets-key -o yaml > sealed-secrets-key.yaml
  # new cluster (before/just after the controller starts), then restart controller
  kubectl apply -f sealed-secrets-key.yaml
  kubectl -n kube-system rollout restart deploy sealed-secrets
  ```
- **Re-seal (no key transfer):** create the raw Secret locally (never committed),
  `kubeseal` it against the new controller, and replace the overlay file:
  ```bash
  kubeseal --controller-name sealed-secrets --controller-namespace kube-system \
    --format yaml < raw-secret.yaml > overlays/dev/app-credentials-sealed.yaml
  ```

> Note: the application Deployments do not currently mount `app-credentials`, so
> the secret's presence/absence does not affect service startup — it exists to
> satisfy the no-plaintext-secrets constraint and the SealedSecrets workflow.

## Image promotion (dev → staging → prod)

`overlays/dev/kustomization.yaml` is updated by the CI bot commit. Staging and
prod overlays pin the same SHA and patch each `ServiceMonitor` to their own
namespace. Promotion = updating the `newTag` in the target overlay and letting
Argo CD sync.

## Teardown

```bash
kind delete cluster --name major-project-cluster
docker rm -f registry
```
