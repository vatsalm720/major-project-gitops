# Validation Guide

Read-only commands to verify each phase is actually working. Everything here is
`get`/`describe`/`logs`/`port-forward` or a local `kustomize` build — no cluster
mutation (except the explicit drift demo in Phase 4, which the assignment allows).

## Phase 1 — containers

```bash
# multi-stage, non-root, labels, healthcheck
docker inspect --format '{{.Config.User}}' localhost:5000/major-project-api-service:bb225dc   # 1000
docker inspect --format '{{json .Config.Labels}}' localhost:5000/major-project-api-service:bb225dc
docker inspect --format '{{json .Config.Healthcheck}}' localhost:5000/major-project-api-service:bb225dc

# compose dependency ordering
docker compose -f ../major-project/docker-compose.yml up -d --wait
docker compose -f ../major-project/docker-compose.yml ps          # all healthy
```

## Phase 2 — Kubernetes

```bash
kubectl get nodes                                   # 3 Ready
kubectl get ns app app-staging app-prod monitoring gitops
kubectl get statefulset,pvc -n app                  # redis 1/1, PVC Bound
kubectl describe hpa api-service -n app             # cpu .../60% + http_requests_per_second .../10 (not <unknown>)
kubectl get pdb -n app                              # ALLOWED DISRUPTIONS = 1 each
kubectl get networkpolicy -n app                    # deny-all + allow-redis + allow-prometheus
kubectl describe resourcequota app-quota -n app     # requests.cpu 2, requests.memory 1Gi
kubectl get vpa -n app chaos-service -o jsonpath='{.spec.updatePolicy.updateMode}'   # Off
```

NetworkPolicy enforcement (negative + positive control):
```bash
# chaos-service is NOT in the redis allow-list -> must time out
CHAOS=$(kubectl get pod -n app -l app=chaos-service -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n app "$CHAOS" -- python3 -c \
 'import socket;socket.create_connection(("redis.app.svc.cluster.local",6379),3)'   # TimeoutError
# worker IS allowed -> must connect
WK=$(kubectl get pod -n app -l app=worker-service -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n app "$WK" -- python3 -c \
 'import socket;s=socket.create_connection(("redis.app.svc.cluster.local",6379),3);s.sendall(b"PING\r\n");print(s.recv(20))'   # +PONG
```

## Phase 3 — CI/CD

- Open a PR against `release` → `pr-validation` runs test/build/trivy/manifest jobs.
- Push to `release` → `release-build` runs build-and-push, load-images,
  update-gitops (bot commit), canary-gate.
- Inspect the bot commit in this repo:
  ```bash
  git log --oneline | grep 'chore(deploy)'
  ```

## Phase 4 — GitOps / Argo CD

```bash
kubectl get applications -n gitops                  # all Synced + Healthy
kubectl get application dev-app -n gitops -o jsonpath='{.status.sync.revision}{"\n"}'

# local render parity
kubectl kustomize overlays/dev >/dev/null && echo dev-ok
kubectl kustomize overlays/staging >/dev/null && echo staging-ok
kubectl kustomize overlays/prod >/dev/null && echo prod-ok
```

Drift demo (the one allowed live edit — Task 21):
```bash
kubectl -n app scale deploy/api-service --replicas=5     # introduce drift
# Argo CD shows OutOfSync; with selfHeal it reverts within ~minutes.
kubectl get application dev-app -n gitops -w
```
[Insert Screenshot: Drift Detection OutOfSync State]

## Phase 5 — observability

```bash
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090 &
# targets up
curl -s 'http://localhost:9090/api/v1/query?query=up{namespace="app"}' | python3 -m json.tool
# recording rule + SLO
curl -s --data-urlencode 'query=job:http_error_rate:ratio_rate5m' http://localhost:9090/api/v1/query
curl -s --data-urlencode 'query=histogram_quantile(0.95, sum(rate(request_duration_seconds_bucket{job="api-service"}[5m])) by (le))' http://localhost:9090/api/v1/query
# rules loaded + healthy
curl -s http://localhost:9090/api/v1/rules | python3 -c 'import sys,json;[print(g["name"]) for g in json.load(sys.stdin)["data"]["groups"]]'
```

Dashboard-as-code:
```bash
kubectl get cm -n monitoring major-project-grafana-dashboard -o jsonpath='{.metadata.labels}'   # grafana_dashboard:"1"
```

## Phase 6 — chaos

See `docs/runbooks/`. Experiment 2 (network partition) has been executed with
recorded observations; Experiments 1 and 3 are procedure-only until run.

## Full read-only health sweep

```bash
kubectl get pods -A | grep -vE 'Running|Completed'   # expect no rows
kubectl get applications -n gitops
```
